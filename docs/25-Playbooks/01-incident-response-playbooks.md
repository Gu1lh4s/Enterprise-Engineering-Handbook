# Incident Response Playbooks

> **Category:** Operations
> **Version:** 1.0.0
> **Level:** Staff Engineer / SRE

---

## Table of Contents

1. [Playbook: Data Breach](#1-playbook-data-breach)
2. [Playbook: DDoS Attack](#2-playbook-ddos-attack)
3. [Playbook: Compromised Credentials](#3-playbook-compromised-credentials)
4. [Playbook: Ransomware](#4-playbook-ransomware)
5. [Playbook: Supply Chain Compromise](#5-playbook-supply-chain-compromise)
6. [Playbook: Database Outage](#6-playbook-database-outage)
7. [Playbook: Production Code Deployment Failure](#7-playbook-production-code-deployment-failure)
8. [Playbook: Phishing Attack](#8-playbook-phishing-attack)
9. [Communication Templates](#9-communication-templates)
10. [Post-Incident Process](#10-post-incident-process)

---

## 1. Playbook: Data Breach

**Severity:** P0 (immediately escalate)
**Trigger:** Unauthorized access to customer data suspected or confirmed

### Detection Signals
- Unusual volume of data exported (> 1000 records in 1h by single user)
- SIEM alert: access to production DB from unexpected IP
- Customer reports seeing another user's data
- Third-party penetration tester or security researcher report
- Bug bounty submission

### Immediate Actions (0-15 minutes)

```bash
# STEP 1: Preserve evidence BEFORE making changes
# Take snapshots, export logs — don't modify until evidence is preserved

# Capture current DB connections
psql -h $DB_HOST -U admin -c "SELECT pid, usename, application_name, client_addr, query_start, query FROM pg_stat_activity WHERE state = 'active';"

# Export audit logs from last 24h (preserve before rotation)
aws logs filter-log-events \
  --log-group-name /prod/api \
  --start-time $(date -d "24 hours ago" +%s000) \
  > /tmp/incident-$(date +%Y%m%d-%H%M%S)-logs.json

# Export DB access logs
aws logs filter-log-events \
  --log-group-name /aws/rds/instance/prod-db/postgresql \
  --filter-pattern '"CONNECT"' \
  --start-time $(date -d "48 hours ago" +%s000) \
  > /tmp/incident-db-connections.json

# STEP 2: Identify the scope
# Which tables were accessed?
# Which tenant IDs / user IDs were exposed?
# Time window of unauthorized access?

# Check CloudTrail for unexpected API calls
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject \
  --start-time $(date -d "24 hours ago" -u +"%Y-%m-%dT%H:%M:%SZ") \
  > /tmp/incident-cloudtrail.json
```

### Containment (15-60 minutes)

```bash
# Option A: Revoke the compromised session/key (if single actor)
# Revoke specific API key
psql -c "UPDATE api_keys SET revoked_at = NOW() WHERE id = 'key_xxx'"

# Invalidate all JWT tokens for affected user (if JWT blacklist enabled)
redis-cli SET "jwt:blacklist:user_xxx" "1" EX 86400

# Option B: If broad compromise — emergency actions
# Block suspicious IPs at WAF
aws wafv2 update-ip-set --scope REGIONAL --name blocked-ips \
  --id $WAF_IP_SET_ID \
  --addresses '["x.x.x.x/32"]' \
  --lock-token $LOCK_TOKEN

# Option C: Nuclear — take service offline (last resort)
kubectl scale deployment api --replicas=0 -n production
```

### Investigation

```bash
# Find all records accessed by the attacker
psql -c "
  SELECT resource_type, resource_id, action, actor_id, ip_address, created_at
  FROM audit_logs
  WHERE ip_address = 'x.x.x.x'
    AND created_at >= '2025-06-27 10:00:00'
  ORDER BY created_at ASC
"

# Identify affected users (for notification)
psql -c "
  SELECT DISTINCT u.id, u.email, u.name
  FROM bookings b
  JOIN users u ON u.id = b.client_id
  WHERE b.tenant_id = 'org_xxx'  -- Potentially compromised tenant
" > /tmp/affected-users.csv

echo "Affected user count: $(wc -l < /tmp/affected-users.csv)"
```

### Notification Requirements

```
GDPR (EU users): 
  → Supervisory authority (DPA): within 72h of becoming aware
  → Data subjects: if high risk to rights and freedoms (no strict deadline but ASAP)
  → Template: See §9 Communication Templates

LGPD (Brazilian users):
  → ANPD: "reasonable timeframe" (guidance suggests 2 business days)
  → Data subjects: if significant risk

PCI-DSS (if payment card data):
  → Card brands (Visa, Mastercard): within 24h
  → Acquiring bank: immediately
  → Forensic investigation required (PFI)

US States (if US users):
  → Most states: 30-90 days (varies by state)
  → California (CCPA): "expedient time"
```

---

## 2. Playbook: DDoS Attack

**Severity:** P0-P1 depending on impact
**Trigger:** Massive traffic spike causing service degradation

### Detection Signals
- Alert: request rate > 10x baseline (WAF metric)
- Alert: error rate > 20% (application)
- CloudFront 502/503 spike
- Customer reports: "site is down"
- Ping timeout from external monitoring

### Response Steps

```bash
# STEP 1: Identify attack type
# Volumetric: massive packet flood → network-level mitigation (AWS Shield)
# Application: L7 HTTP flood → rate limiting, WAF rules
# Amplification: DNS/NTP reflection → contact upstream provider

# Check traffic pattern
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --period 60 \
  --statistics Sum \
  --start-time $(date -d "2 hours ago" -u +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Check WAF for blocked requests
aws wafv2 get-sampled-requests \
  --web-acl-arn $WAF_ACL_ARN \
  --rule-metric-name ALL \
  --scope CLOUDFRONT \
  --time-window StartTime=$(date -d "1 hour ago" +%s),EndTime=$(date +%s) \
  --max-items 100

# STEP 2: Enable AWS Shield Advanced (if not already)
# STEP 3: Tighten rate limiting
# Temporary: block entire country if attack is geo-localized

# WAF: add emergency rate limit rule (100 req/5min per IP)
aws wafv2 update-rule-group ...

# STEP 4: Enable CloudFront caching for all public endpoints
# STEP 5: Scale up if legitimate traffic mixed in
kubectl scale deployment api --replicas=20 -n production
kubectl autoscaler set-max replicas=50 api -n production

# STEP 6: Engage AWS DDoS Response Team (DRT) if AWS Shield Advanced
aws shield create-protection-group ...
```

---

## 3. Playbook: Compromised Credentials

**Severity:** P0 (admin/privileged) | P1 (regular user)
**Trigger:** Credentials found in breach dump, suspicious login detected, HaveIBeenPwned alert

### Admin/Privileged Account Compromise

```bash
# STEP 1: IMMEDIATE — revoke all access
# Disable SSO account (kills all active sessions via SAML/OIDC)
# In Okta/Azure AD: deactivate user, force sign-out all sessions

# Revoke all API keys
psql -c "UPDATE api_keys SET revoked_at = NOW() WHERE user_id = 'compromised_user_id'"

# Remove SSH keys
cat ~/.ssh/authorized_keys | grep -v "compromised_key_fingerprint" > /tmp/auth_keys_clean
mv /tmp/auth_keys_clean ~/.ssh/authorized_keys

# Revoke AWS credentials if using long-lived keys (should not exist)
aws iam delete-access-key --access-key-id $COMPROMISED_KEY_ID

# STEP 2: Audit actions taken with compromised credentials
# CloudTrail: all API calls by this user/key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=$COMPROMISED_USER \
  --start-time $(date -d "30 days ago" -u +"%Y-%m-%dT%H:%M:%SZ")

# Application audit log
psql -c "
  SELECT * FROM audit_logs
  WHERE actor_id = 'compromised_user_id'
  ORDER BY created_at DESC
  LIMIT 500
"

# STEP 3: Determine blast radius
# What resources could they access?
# What did they actually access (from logs)?
# Any data exfiltrated?

# STEP 4: Credential rotation (all systems they had access to)
# STEP 5: Root cause (phishing? credential stuffing? weak password? breach?)
```

---

## 4. Playbook: Ransomware

**Severity:** P0 (existential threat)
**Trigger:** Files encrypted, ransom note found, antivirus alert, operations halted

### CRITICAL: DO NOT PAY RANSOM FIRST — investigate options

```bash
# STEP 1: ISOLATE IMMEDIATELY
# Disconnect all affected systems from network
# Kill production database connections
# Take snapshots before encryption spreads

# In Kubernetes: cordon affected nodes (prevent new pods)
kubectl cordon affected-node-1 affected-node-2

# AWS: isolate affected EC2 instances (security group with no ingress/egress)
aws ec2 modify-instance-attribute --instance-id $INSTANCE_ID \
  --groups $ISOLATION_SG_ID  # SG with no rules

# RDS: snapshot immediately (before encryption potentially reaches DB)
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres \
  --db-snapshot-identifier ransomware-incident-$(date +%Y%m%d)

# STEP 2: Preserve evidence
# Memory dump (if attackers still active)
# Do NOT reboot — losing in-memory evidence
# Call forensics firm if uncertain

# STEP 3: Assess recovery options
# Option A: Restore from clean backup (preferred)
# Option B: Decrypt (if decryption tool exists — check nomoreransom.org)
# Option C: Pay ransom (last resort; no guarantee; fund criminals)

# STEP 4: Restore from backup
# Identify last clean backup (before infection — attackers may lurk weeks)
aws rds describe-db-snapshots --db-instance-identifier prod-postgres \
  --query "DBSnapshots[?SnapshotCreateTime<='2025-06-20T00:00:00']|[0]"

# Restore to new instance (not same — old may be compromised)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier ransomware-incident-YYYYMMDD \
  --db-instance-class db.r6g.large

# STEP 5: Rebuild infrastructure from IaC (Terraform/Pulumi)
# Don't reuse potentially compromised machines
# Deploy fresh from git (verify git repo not compromised)
terraform init && terraform apply -var-file=prod.tfvars

# STEP 6: Law enforcement
# Report to local police and national cybercrime unit
# FBI IC3 (USA): ic3.gov
# CERT.br (Brazil): cert.br
```

---

## 5. Playbook: Supply Chain Compromise

**Severity:** P0 (immediate)
**Trigger:** Malicious code in dependency detected, Typosquatting discovered, CI compromise

### Compromised NPM Package

```bash
# STEP 1: Identify what was running the malicious package
grep -r "malicious-package" package-lock.json
# Check all services that depend on it
git log --all -- "package-lock.json" | head -20

# STEP 2: Check if deployed to production
kubectl get deployments -n production -o jsonpath='{.items[*].spec.template.spec.containers[*].image}'
# Find which image versions used the bad package

# STEP 3: Check what the malicious package could do
# Read the malicious code (if available)
# Common payloads: credential theft, backdoor, data exfiltration

# STEP 4: Rotate all credentials (assume exfiltrated)
# - Database passwords
# - API keys
# - JWT secrets
# - Cloud credentials (AWS, GCP)
# - SSO credentials

# STEP 5: Check for active backdoors
# Look for new processes, unusual cron jobs, reverse shells
ps aux | grep -v "expected-processes"
netstat -tlnp | grep LISTEN
crontab -l

# STEP 6: Rebuild and redeploy
npm audit fix
# Pin to known-good version:
# "malicious-package": "0.9.99"  ← last known good

# Rebuild Docker images from scratch (no cache)
docker build --no-cache -t api:cleaned .

# STEP 7: Pin all dependencies going forward
# Implement npm ci (uses exact lock file)
# Enable Dependabot security alerts
# Require SHA pinning for GitHub Actions
```

---

## 6. Playbook: Database Outage

**Severity:** P0 (production down) | P1 (degraded)
**Trigger:** Database unreachable, all requests failing with connection error

```bash
# STEP 1: Confirm outage
psql -h $DB_HOST -U admin -c "SELECT 1" -c "SELECT NOW()"
# Test from within Kubernetes network:
kubectl run -it --rm debug --image=postgres:16 --restart=Never -- \
  psql -h postgres-service -U admin -c "SELECT 1"

# STEP 2: Check RDS status
aws rds describe-db-instances --db-instance-identifier prod-postgres \
  --query "DBInstances[0].DBInstanceStatus"

# Check RDS events
aws rds describe-events --source-identifier prod-postgres --duration 60

# STEP 3: Check PostgreSQL logs
kubectl logs postgres-0 -n production --since=30m | tail -100

# STEP 4: Common causes and fixes

# A: Connection pool exhausted (too many connections)
psql -c "SELECT count(*) FROM pg_stat_activity"
# Fix: restart app pods to release connections
kubectl rollout restart deployment/api -n production
# Long-term: add PgBouncer

# B: Disk full
kubectl exec -it postgres-0 -- df -h /var/lib/postgresql/data
# Fix: expand PVC or purge old data

# C: Long-running query blocking others
psql -c "SELECT pid, query, query_start FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start ASC"
# Kill blocking query (use carefully):
psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid = BLOCKING_PID"

# D: Primary failover (Aurora/RDS Multi-AZ)
# Wait for automatic failover (60-120 seconds typically)
# Or manual failover:
aws rds failover-db-cluster --db-cluster-identifier prod-cluster

# E: Corruption / crash
# Check pg_log for PANIC or FATAL messages
kubectl logs postgres-0 | grep -E "PANIC|FATAL|corruption"
# May need to restore from backup

# STEP 5: Enable read replica for read traffic during primary recovery
# Update DNS CNAME or connection string to point to read replica
# Disable write operations temporarily (feature flag)
```

---

## 7. Playbook: Production Code Deployment Failure

**Severity:** P0 (all users affected) | P1 (degraded)
**Trigger:** Post-deployment error spike, health checks failing, user reports broken features

```bash
# STEP 1: Detect and confirm
kubectl rollout status deployment/api -n production  # Is rollout stuck?
kubectl get pods -n production  # Are pods crashing (CrashLoopBackOff)?

# Check error rate (immediately after deploy):
# Prometheus: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05

# STEP 2: Rollback (first action — do not investigate while users are impacted)
# Option A: Kubernetes rollback
kubectl rollout undo deployment/api -n production
kubectl rollout status deployment/api -n production  # Wait for rollback to complete

# Option B: Deploy specific previous version
kubectl set image deployment/api api=ghcr.io/gu1lh4s/api:PREVIOUS_SHA -n production

# Option C: Revert via Helm
helm rollback api -n production  # Rolls back to previous Helm release

# STEP 3: Verify recovery
kubectl get pods -n production  # All Running?
curl https://api.example.com/health  # Health check passing?
# Check error rate in Grafana: should drop to < 0.1%

# STEP 4: Investigate the bad commit (after users are recovered)
git log --oneline HEAD~5..HEAD  # Recent commits
git show HEAD  # What changed?

# STEP 5: Root cause
# Run locally with production-like config
# Check: new environment variables required but not set?
# Check: database migration not applied before code deploy?
# Check: breaking change in API that client wasn't updated for?

# STEP 6: Fix forward
# Fix the issue in the code
# Test in staging (run production-like smoke tests)
# Deploy to production with extra monitoring
# Watch for 30 minutes before calling it stable
```

---

## 8. Playbook: Phishing Attack

**Severity:** P1 (credential harvested) | P2 (reported, no click)
**Trigger:** Employee reports suspicious email, credential used from unusual location

```bash
# STEP 1: Identify scope
# Did anyone click the link?
# Did anyone enter credentials?
# Which employees received the email?

# Check email gateway logs for recipients
# (Google Workspace admin → Reports → Email logs)

# STEP 2: If credentials entered → treat as Compromised Credentials (Playbook 3)

# STEP 3: Block the phishing domain/IP
# Add to email gateway blocklist
# Add to DNS blocklist (Pi-hole, NextDNS)

# STEP 4: Alert all employees who received it
EMAIL_SUBJECT="[Security Alert] Phishing Email in Your Inbox"
# See Communication Templates §9

# STEP 5: Check security awareness training completion
# Who hasn't completed phishing awareness training?
# Schedule mandatory training for those who clicked

# STEP 6: Report to relevant authorities
# Google/Microsoft: report phishing URL to safe browsing
# CERT.br (Brazil): abuse.br
# Anti-Phishing Working Group: reportphishing@apwg.org

# STEP 7: Collect indicators of compromise
PHISHING_IOC = {
  sender_email: "noreply@fake-example.com",
  sender_ip: "x.x.x.x",
  phishing_url: "https://evil-example.com/login",
  subject: "Urgent: Verify your account",
  sha256_of_attachment: "abc123..."
}
# Share IOCs with threat intel team and industry peers
```

---

## 9. Communication Templates

### Internal: Incident Declaration

```
Subject: [SECURITY INCIDENT P0] Data breach suspected - INC-2025-001

@here Declaring a P0 security incident.

INCIDENT: INC-2025-001
SEVERITY: P0 (Critical)
IC (Incident Commander): @your-name
BRIDGE: [Zoom/Meet link]
STATUS: Investigating

SUMMARY:
[1-2 sentences: what happened, what we know so far]

IMMEDIATE IMPACT:
[Who is affected, what services, how many users]

ACTIONS TAKEN:
[ ] Logs preserved
[ ] Attack source identified
[ ] Containment action taken

NEXT UPDATE: [Time — 30 min for P0]

All engineers: join bridge now. Non-essential communication on #incident-2025-001 channel.
```

### Customer: Data Breach Notification (GDPR-compliant)

```
Subject: Important Security Notice — [Company Name]

Dear [Name],

We are writing to inform you about a security incident that affected your account on [Platform Name].

WHAT HAPPENED:
On [Date], we discovered that an unauthorized party gained access to [brief description of data accessed].

WHAT INFORMATION WAS INVOLVED:
The following information may have been accessed:
• [Name]
• [Email address]
• [Other specific data types]

Note: Your payment information was NOT affected. We do not store full payment card numbers.

WHAT WE DID:
• Immediately revoked unauthorized access upon discovery
• Implemented additional security controls
• Notified the relevant data protection authority

WHAT YOU SHOULD DO:
• Change your password on [Platform Name]
• If you use the same password elsewhere, change it there too
• Be alert for suspicious emails claiming to be from us

We sincerely apologize for this incident. If you have questions, please contact our Data Protection Officer at dpo@example.com or call [number].

Sincerely,
[Name], CEO
[Company Name]
```

### Regulator: GDPR 72h Notification Template

```
To: [Supervisory Authority email — e.g., complaints@ico.org.uk]
Subject: Personal Data Breach Notification — [Company Name] — [Date]

Organization: [Company Name]
DPO Contact: [Name] — dpo@example.com — [Phone]
Report Number: [Internal Reference]
Date of Discovery: [Date and Time UTC]
Date of This Notification: [Date — must be within 72h]

1. NATURE OF THE BREACH
[Description: unauthorized access, accidental disclosure, etc.]

2. CATEGORIES AND APPROXIMATE NUMBER OF DATA SUBJECTS
Categories: [Names, emails, phone numbers, etc.]
Approximate number: [X] data subjects

3. CATEGORIES AND APPROXIMATE NUMBER OF RECORDS
[X] records across [Y] database tables

4. LIKELY CONSEQUENCES
[Risk assessment: reputational, financial, identity theft risk to subjects]

5. MEASURES TAKEN OR PROPOSED
Immediate:
  - Revoked unauthorized access at [time]
  - Implemented [specific controls]
  
Longer-term:
  - [Remediation actions with timeline]

6. CONTACT FOR FURTHER INFORMATION
[DPO name and contact details]

We will provide a follow-up report within [X] days with full root cause analysis.
```

---

## 10. Post-Incident Process

### Blameless Post-Mortem Template

```markdown
# Post-Mortem: [Incident Title]

**Incident ID:** INC-2025-001
**Date:** 2025-06-27
**Duration:** 2h 15m (10:30 - 12:45 UTC)
**Severity:** P0
**Status:** Resolved
**Post-mortem written by:** [Name]
**Post-mortem reviewed by:** [Team]
**Review meeting:** 2025-06-29 14:00 UTC

## Impact
- Users affected: ~1,200
- Revenue lost: ~R$15,000 (estimated 2h outage × avg revenue/hour)
- SLO breach: 99.85% for the month (SLO: 99.9%)
- Error budget consumed: 45 minutes of our 43.8 minute monthly budget

## Timeline (all times UTC)
| Time  | Event |
|-------|-------|
| 10:28 | Deploy api:1.3.0 started |
| 10:31 | Error rate spike detected (Grafana alert) |
| 10:33 | On-call engineer paged (PagerDuty) |
| 10:38 | Incident Commander declares P0 |
| 10:45 | Root cause identified: missing env var DATABASE_POOL_SIZE |
| 10:52 | Rollback decision made |
| 11:02 | Rollback complete — error rate recovering |
| 12:45 | Incident closed, post-mortem scheduled |

## Root Cause
The deployment of api:1.3.0 introduced a new required environment variable
`DATABASE_POOL_SIZE` that was not added to the production Kubernetes secret.
The application started with `undefined` pool size, which defaulted to 0,
causing all database connections to fail immediately.

## Contributing Factors
1. No staging environment parity with production secrets
2. No pre-deployment checklist that validates required env vars
3. Deployment happened at peak hours (10:30 UTC = 07:30 BRT)
4. Rollback decision was delayed 7 minutes (unclear ownership)

## What Went Well
- PagerDuty alert fired within 3 minutes of first error
- Incident Commander was designated immediately
- Rollback was clean and successful
- Customer communication was drafted within 30 minutes

## What Went Poorly
- Missing env var not caught in CI or staging
- 7 minute delay in rollback decision (15 min acceptable for P0)
- Status page not updated until 11:15 (25 min lag)
- Incident bridge was noisy — too many engineers

## Action Items
| Action | Owner | Due | Priority |
|--------|-------|-----|----------|
| Add env var validation to deployment script (fail if required vars missing) | [Engineer] | 2025-07-04 | High |
| Add required env var list to deploy runbook | [DevOps] | 2025-07-04 | High |
| Staging → production secret parity check in CI | [DevOps] | 2025-07-11 | Medium |
| Define rollback decision authority in incident playbook | [SRE Lead] | 2025-07-07 | High |
| Automate status page update (link to AlertManager) | [Infra] | 2025-07-18 | Medium |
| Schedule deployments outside business hours for major releases | [Tech Lead] | Policy | Low |

## Lessons Learned
The incident reveals a gap in our deployment validation: we validate code correctness
(tests, type check) but not configuration completeness. Adding a "preflight check"
that validates all required env vars exist before traffic is shifted would have
prevented this outage entirely.
```
