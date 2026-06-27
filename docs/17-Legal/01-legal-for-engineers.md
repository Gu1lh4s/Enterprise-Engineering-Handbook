# Legal Fundamentals for Engineers

> **Category:** Legal / Compliance
> **Version:** 1.0.0
> **Level:** All Engineers, especially Tech Leads and Staff Engineers

> **Disclaimer:** This is educational reference material, not legal advice.
> For actual legal decisions, consult your company's legal counsel.

---

## Table of Contents

1. [Open Source Licenses](#1-open-source-licenses)
2. [Intellectual Property Basics](#2-intellectual-property-basics)
3. [Software Contracts Engineers Touch](#3-software-contracts-engineers-touch)
4. [Data Processing Agreements (DPA)](#4-data-processing-agreements-dpa)
5. [SLAs and SLOs](#5-slas-and-slos)
6. [Employee IP Assignment](#6-employee-ip-assignment)
7. [Open Source Contribution Policy](#7-open-source-contribution-policy)
8. [License Compliance in Practice](#8-license-compliance-in-practice)

---

## 1. Open Source Licenses

Understanding licenses matters because using the wrong one can legally obligate your company to open-source its proprietary code.

### License Spectrum

```
PERMISSIVE ─────────────────────────────────────────── COPYLEFT
     │                    │                   │              │
    MIT               Apache 2.0           LGPL 2.1        GPL 3.0
    BSD               MPL 2.0              EUPL             AGPL 3.0

Permissive: Use freely, keep attribution, no obligation to open-source your code
Copyleft: If you distribute derivative works, you must release under the same license
```

### License Quick Reference

| License | Can Use Commercially | Must Open Source | Patent Grant | Trademark Protection | Copyleft Scope |
|---------|---------------------|-----------------|--------------|---------------------|----------------|
| MIT | ✅ | ❌ | ❌ | ❌ | None |
| BSD 2/3-Clause | ✅ | ❌ | ❌ | ✅ (3-clause) | None |
| Apache 2.0 | ✅ | ❌ | ✅ | ✅ | None |
| MPL 2.0 | ✅ | Modified files only | ✅ | ✅ | File-level |
| LGPL 2.1 | ✅ (as library) | LGPL code only | ❌ | ❌ | Library only |
| GPL 3.0 | ✅ (internally) | ✅ if distributed | ✅ | ❌ | Full project |
| AGPL 3.0 | ❌ safely | ✅ if served via network | ✅ | ❌ | Full project + network |

### Key Licenses Explained

```
MIT:
  Most permissive. Include copyright notice and MIT license text in your product.
  That's it. You can use it in proprietary software.
  Risk: No patent protection. If the author has patents covering the library,
  they could (in theory) sue you separately.

Apache 2.0:
  Like MIT but adds: explicit patent license + protection against CLAs.
  Contributor grants you a patent license for the code they contribute.
  Best "safe" choice for dependencies in commercial products.
  NOTICE file required if the package has one.
  
  Compatibility: Apache 2.0 code CAN go into MIT products (just add NOTICE).
  MIT code CAN go into Apache 2.0 products (MIT is more permissive).

LGPL:
  Designed for libraries. If you use the LGPL library dynamically (link at runtime)
  and don't modify the library source, you can keep your code proprietary.
  If you statically link (embed) or modify the library, you must open-source those parts.
  
  Safe use: dynamic linking, no modification → use in proprietary software
  Risky use: static embedding, modification → legal exposure

GPL:
  If your software includes GPL code and you distribute the software externally,
  your entire codebase must be licensed under GPL.
  
  Internal use only (never distributed): GPL is safe internally.
  SaaS/cloud service: distributing the service, not the software → GPL doesn't apply
    UNLESS the license is AGPL (see below).
  
  Bottom line: Never add GPL code to a proprietary product that's distributed to users.

AGPL:
  Like GPL, but the "distribution" trigger includes making software available over a network.
  If your SaaS uses AGPL libraries, you may be obligated to open-source your product.
  
  Policy for most commercial companies: AGPL is BANNED from commercial products.
  Check with legal before using ANY AGPL dependency.

Creative Commons (CC):
  For content, not software. Do NOT use CC licenses for code.
  CC BY, CC BY-SA: fine for documentation, not for code.
  CC BY-NC: prohibits commercial use — generally banned in commercial settings.
```

### License Compatibility Matrix

```
Can MIT code be incorporated into...
  Apache 2.0 project? YES (MIT is permissive)
  GPL project?        YES (MIT is compatible with GPL)
  Proprietary?        YES

Can Apache 2.0 code be incorporated into...
  MIT project?        YES (add NOTICE file)
  GPL v3 project?     YES (Apache 2.0 is GPL v3 compatible)
  GPL v2 project?     NO (Apache 2.0 and GPL v2 are incompatible)
  Proprietary?        YES

Can GPL v3 code be incorporated into...
  Apache 2.0 project? NO (GPL is more restrictive)
  MIT project?        NO (GPL is more restrictive)
  Proprietary?        NO (must release entire product under GPL)
  AGPL project?       YES (both copyleft)
```

---

## 2. Intellectual Property Basics

```
Types of IP relevant to software:

Copyright:
  What it protects: Expression (specific source code, documentation, UI designs)
  What it doesn't protect: Ideas, algorithms, functionality, APIs
  When it arises: Automatically when created — no registration required
  Duration: Life of author + 70 years (corporate: 95 years from publication)
  Relevance: Your code is copyrighted; you must have rights to use others' code

Trade Secrets:
  What it protects: Confidential business information (algorithms, business logic, source code)
  Protection condition: Must be kept secret (NDAs, access controls)
  Duration: Indefinite (as long as kept secret)
  Lost when: Publicly disclosed (even accidentally)
  Relevance: Proprietary source code is a trade secret — treat accordingly

Patents:
  What it protects: Inventions (processes, methods, sometimes software)
  When it arises: Only after application and grant (18 months minimum)
  Duration: 20 years from filing date
  Relevance: Patent trolls target software; Apache 2.0 / MIT provide no patent defense

Trademark:
  What it protects: Brand identifiers (names, logos, "look and feel")
  Relevance: Don't use open-source project names in your product name without checking

The key employee IP question:
  Work made for hire: Code written during employment, on company time, or using company
  resources is owned by the EMPLOYER, not the employee.
  
  Most employment contracts contain IP assignment clauses — read yours.
  Side projects: check if your contract requires you to disclose them.
```

---

## 3. Software Contracts Engineers Touch

### NDA (Non-Disclosure Agreement)

```
What engineers encounter:
  - Pre-sales technical discussions with prospects
  - Vendor evaluation discussions
  - Partnership discussions

What NDAs typically say:
  - Don't disclose confidential information shared during discussion
  - Confidential info can only be used for the agreed purpose
  - Exclusions: publicly available info, info you already knew

Engineer implications:
  - Don't share proprietary code/architecture with prospects without checking if NDA is in place
  - Don't use prospect's confidential data to build features without permission
  - Receiving confidential info from vendor = you're now bound to protect it
```

### SaaS Subscription Agreement / ToS

```
What engineers should understand:
  - Acceptable Use Policy (AUP): What users can/cannot do with your platform
  - Data handling: What you promise about user data (support your privacy policy)
  - Uptime SLA: If you commit to 99.9%, engineering owns that target
  - Security commitments: "SOC 2 Type II" or "ISO 27001" in ToS means you need those certs
  - Data portability / deletion: GDPR data subject rights become contractual obligations

Developer mistake:
  Telling customers "yes we're SOC 2 compliant" without checking with legal.
  If it's in the contract, you're legally bound to it.
```

### Software License Agreement (for your product)

```
Licenses customers sign for your commercial software:
  Perpetual license: Customer can use the software forever (even if they stop paying)
  Subscription license: Usage rights tied to ongoing payment
  Named user / concurrent user / site license: Different metrics
  
  Engineering concern: License enforcement mechanisms
    - Seat counting (users in your system)
    - Feature gating (paid-tier functionality)
    - Usage metering (API calls, storage)
  
  These need to be built correctly — contract disputes often start with metering bugs
```

---

## 4. Data Processing Agreements (DPA)

```
GDPR/LGPD require a DPA between:
  Controller (decides WHY data is processed) and
  Processor (does the processing on controller's behalf)

When you're a controller:
  You need DPAs with your vendors who process your users' data:
  - AWS, GCP, Azure (cloud infrastructure)
  - SendGrid/Mailgun (email)
  - Twilio (SMS)
  - Stripe (payments — PCI + GDPR)
  - Datadog, Sentry (observability — they see your users' data in logs)
  - Any AI provider (OpenAI, Anthropic) if you send user data in prompts
  
  Most cloud providers provide standard DPAs — sign them before going live.

What a DPA must contain (GDPR Art. 28):
  ✓ Process data only on documented instructions
  ✓ Ensure confidentiality (staff bound by NDA)
  ✓ Implement appropriate security (Art. 32)
  ✓ Sub-processors: list them, get controller approval for new ones
  ✓ Data subject rights: help controller fulfill them
  ✓ At end of service: return or delete data
  ✓ Audit rights: allow controller to audit/inspect

Engineer implications:
  Before integrating a new vendor that will process user data:
  1. Check if a DPA exists with this vendor
  2. If not, get legal to execute one before you go live
  3. Document the vendor in your Record of Processing Activities (RoPA)
  4. Check if they sub-process in non-adequate countries (SCCs may be needed)
```

---

## 5. SLAs and SLOs

```
SLA (Service Level Agreement):
  Contractual commitment to customers.
  Contains: uptime target, response time, remedies (credits, termination rights).
  Violation has financial/legal consequences.

SLO (Service Level Objective):
  Internal engineering target.
  Should be stricter than SLA to create a safety buffer.
  
  Example:
    SLA (customer-facing): 99.9% uptime monthly
    SLO (internal target): 99.95% uptime monthly
    Error budget (monthly): 4.38 hours (SLA); 2.19 hours (SLO)

Common SLA tiers:
  99.9%   = 8.76 hours/year downtime   (starter / standard tier)
  99.95%  = 4.38 hours/year downtime   (professional tier)
  99.99%  = 52.6 minutes/year downtime (enterprise / critical)
  99.999% = 5.26 minutes/year downtime (mission-critical, e.g., payments)

What engineers need to know about SLAs:
  1. You're building to support the SLA — know what yours is
  2. Planned maintenance: usually excluded from SLA calculation
     → Communicate maintenance windows to customers in advance
  3. Customer-caused issues usually excluded (their integration broke, not your service)
  4. Credits don't make customers whole for real damages — set SLA at achievable level
  
Measurement:
  SLA uptime calculation:
    Uptime % = ((minutes in month - minutes down) / minutes in month) × 100
    June 2026: 43,200 minutes
    99.9% = max 43.2 minutes down
  
  What counts as "down":
    Define this precisely in the SLA (usually: health check endpoint unreachable from 3+ regions)
    Partial degradation (some endpoints slow): usually not full "downtime" but may have separate SLA
```

---

## 6. Employee IP Assignment

```
What it means:
  When you join a company, your employment agreement typically includes a clause stating
  that any IP you create during employment that relates to the company's business
  belongs to the company, not to you.

What's typically included:
  ✓ Code written during work hours
  ✓ Code written on personal time but for company projects
  ✓ Code written using company resources (laptop, cloud accounts)
  ✓ Inventions related to company's business or research

What's typically EXCLUDED (in engineer-friendly agreements):
  ✗ Personal projects unrelated to company's business
  ✗ Open source contributions (usually — check your specific agreement)
  ✗ Projects started before joining (document prior inventions in the agreement)

Engineer action items when joining:
  1. READ the IP assignment clause before signing
  2. If you have existing side projects, declare them as "prior inventions" at signing
  3. Ask for clarification on open-source contribution policy
  4. Keep personal and work projects clearly separated (different machines, accounts, hours)

Open source contributions as an employee:
  Contributing to open source on personal time:
    → Usually permitted if the project is unrelated to your employer's core business
    → Some companies require you to get approval first (especially large tech companies)
    → If it's the same tech stack your employer uses: get written approval
  
  Contributing on work time:
    → Employer usually owns the contribution unless they have an OSS policy
    → Need company sign-off; they may need to sign a CLA as the IP holder
```

---

## 7. Open Source Contribution Policy

```
As a company, when your engineers contribute to open source:

CLA (Contributor License Agreement):
  Many open source projects (React, Kubernetes, Linux, Apache) require contributors
  to sign a CLA before accepting contributions.
  
  Individual CLA: engineer signs personally (they own the contribution)
  Corporate CLA: company signs (company grants rights to the project)
  
  When to use corporate CLA:
    If the contribution was made on company time or using company resources,
    the company owns the IP → company must sign the CLA, not the individual.
    
  Process:
    1. Engineer wants to contribute to Project X
    2. Check if Project X has a CLA requirement
    3. If yes: legal signs corporate CLA (one-time per project, covers all employees)
    4. Engineer contributes; CLA is on file

DCO (Developer Certificate of Origin):
  Alternative to CLA (used by Linux kernel, many CNCF projects).
  Sign-off line in commit: "Signed-off-by: Name <email>"
  Certifies: you have the right to submit this code under the project's license.
  
  Corporate DCO: use company email address in the sign-off.

What your engineers need to do:
  1. Check if the project has CLA or DCO requirement
  2. For CLA: get legal to sign the corporate CLA before submitting contribution
  3. For DCO: use company email in git config for work contributions
  4. Never submit code that includes company trade secrets or proprietary algorithms
  5. Don't release dependencies that aren't already public (e.g., don't accidentally release internal libs)
```

---

## 8. License Compliance in Practice

### Dependency Scanning

```bash
# Check all licenses in a Node.js project
npx license-checker --summary
npx license-checker --excludePackages "project@1.0.0" --json > licenses.json

# Check for copyleft (GPL, AGPL) dependencies
npx license-checker --excludePackages "project@1.0.0" --failOn "GPL;AGPL"

# Python projects
pip-licenses --format=table --with-authors

# Go projects
go-licenses save ./... --save_path=./licenses
```

```typescript
// Automate in CI — block merges with copyleft dependencies
// .github/workflows/license-check.yml
name: License Check
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - name: Check licenses
        run: |
          npx license-checker \
            --failOn "GPL;AGPL;EUPL;CC-BY-SA" \
            --excludePrivatePackages \
            --summary
```

### NOTICE file compliance (Apache 2.0)

```bash
# Generate NOTICE file for Apache 2.0 dependencies
# Required: include copyright notices from NOTICE files of Apache 2.0 deps

# Node.js: extract NOTICE content from node_modules
find node_modules -name "NOTICE*" -exec echo "--- {} ---" \; -exec cat {} \; > THIRD_PARTY_NOTICES.txt

# Alternatively: use license-checker to generate attributions
npx license-checker --json | node scripts/generate-notices.js > THIRD_PARTY_NOTICES.txt
```

### License Policy (suggested defaults)

```
APPROVED (use without review):
  MIT, BSD 2-Clause, BSD 3-Clause, Apache 2.0, ISC, CC0, Unlicense
  
REVIEW REQUIRED (legal must approve before use):
  LGPL 2.1, LGPL 3.0, MPL 2.0, CDDL, EUPL
  (acceptable in some cases — dynamic linking, no modification)
  
BANNED (never use in proprietary products without explicit legal sign-off):
  GPL 2.0, GPL 3.0, AGPL 3.0, SSPL (Server Side Public License)
  CC BY-SA (for code), CC BY-NC (prohibits commercial use)
  
UNKNOWN / CUSTOM:
  Always review with legal before using. No license = all rights reserved (do not use).

For internal tools / scripts (not distributed to customers):
  GPL is acceptable (GPL triggered by distribution, not internal use).
  AGPL is still risky for internal web services.
```
