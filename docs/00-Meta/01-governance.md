# Engineering Handbook Governance

> **Category:** Meta / Governance
> **Version:** 1.0.0
> **Level:** All Engineers

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [How to Use This Handbook](#2-how-to-use-this-handbook)
3. [Contributing Guidelines](#3-contributing-guidelines)
4. [Version Control and Change Management](#4-version-control-and-change-management)
5. [ADR Process](#5-adr-process)
6. [Review and Currency](#6-review-and-currency)
7. [Enforcement and Compliance](#7-enforcement-and-compliance)

---

## 1. Purpose and Scope

This handbook is the **single authoritative source** for engineering standards at this organization. It exists to:

- Give engineers a reference for production-quality decisions without having to research from scratch
- Reduce inconsistency across services, teams, and codebases
- Capture institutional knowledge that would otherwise live in Slack history or individual heads
- Provide onboarding material for new engineers at all levels

### What this is NOT

- A tutorial — engineers are expected to be competent in their stack; this is a reference for doing it *correctly* in production
- The only way to do things — reasonable deviations require a documented reason (ADR or PR comment)
- A substitute for judgment — these are default best practices, not rules for every situation

### Scope

Applies to all software produced by this engineering organization: backend services, frontend applications, infrastructure, pipelines, tooling.

---

## 2. How to Use This Handbook

### Navigation

```
docs/
├── 00-Meta/          → How the handbook works
├── 01-Engineering/   → SOLID, DDD, Clean Code, Clean Architecture
├── 02-Architecture/  → System design, distributed systems, event-driven
├── 07-Security/      → SQL injection, XSS, authentication
├── 08-DevSecOps/     → CI/CD security, SAST/DAST, supply chain
├── 09-Cloud/         → AWS patterns (IAM, VPC, ECS, RDS)
├── 09-Database/      → PostgreSQL, indexing, migrations
├── 10-Backend/       → Node.js production patterns
├── 11-Frontend/      → React architecture, testing, a11y
├── 12-API/           → REST, GraphQL, gRPC, webhooks
├── 13-AI/            → LLMs, RAG, Agents, Evals
├── 13-Testing/       → Testing pyramid, TestContainers, k6
├── 14-Docker/        → Production Dockerfiles, security
├── 15-Kubernetes/    → Deployment, HPA, NetworkPolicy, Kyverno
├── 16-Observability/ → Prometheus, Pino, OTel, SLOs
├── 17-Legal/         → Licenses, contracts, IP
├── 17-Performance/   → Profiling, caching, optimization
├── 18-LGPD/          → LGPD compliance implementation
├── 19-GDPR/          → GDPR compliance implementation
├── 20-PCI/           → PCI-DSS requirements
├── 21-ISO/           → ISO 27001 ISMS
├── 22-SaaS/          → Multi-tenancy, RBAC, Stripe
├── 23-OWASP/         → OWASP Top 10 with code examples
├── 24-NIST/          → NIST CSF 2.0, SP 800-53, AI RMF
├── 25-Playbooks/     → Incident response playbooks
├── 26-Checklists/    → Pre-deploy, security, migration, launch
└── 26-Templates/     → PRD, RFC, design doc templates
```

### Quick Reference

| Question | Go to |
|----------|-------|
| "How should I design this API?" | `12-API/01-api-design.md` |
| "Is this PostgreSQL query safe to run?" | `09-Database/01-postgresql-patterns.md` § Migration |
| "What security should this endpoint have?" | `07-Security/` + `26-Checklists/#3-security-review` |
| "How should this service be deployed?" | `15-Kubernetes/01-kubernetes-production.md` |
| "How do I write a proper Dockerfile?" | `14-Docker/01-docker-production.md` |
| "What do I need for GDPR compliance?" | `19-GDPR/01-gdpr-implementation.md` |
| "We're having an incident, what do?" | `25-Playbooks/01-incident-response-playbooks.md` |
| "Pre-deploy, what should I check?" | `26-Checklists/01-master-checklists.md` §1 |

---

## 3. Contributing Guidelines

### When to contribute

- You find a mistake or outdated information → fix it immediately
- You encounter a production problem not covered here → add the solution
- Your team builds a novel pattern that others should know about → document it
- You find a better approach than what's documented → propose it with evidence

### Process for changes

```
Small fix (typo, code correction, minor update):
  → PR directly, one approver, merge same day

Standard update (new section, updated pattern):
  → PR + description of why the change improves the handbook
  → Review by one senior engineer in the relevant domain
  → Merge after approval

Breaking change (deprecate existing guidance, new mandatory standard):
  → Discussion first (engineering meeting or Slack + 48h async review)
  → PR with "BREAKING" in title
  → Two senior engineer approvals
  → Update CHANGELOG.md
  → Communication to all engineers
```

### Writing standards

```
Style rules for contributions:
  - Target audience: capable engineers who want production reference, not tutorials
  - Show real code: production-quality, not simplified examples
  - No filler: every sentence earns its place
  - Code blocks: runnable and tested when possible
  - Commands: tested on current OS (note if OS-specific)
  - Links: absolute URLs only (relative links break when docs move)

Required sections for new documents:
  - Frontmatter (Category, Version, Level)
  - Table of Contents (for docs > 200 lines)
  - At least one working code example per major section
  
Optional but valuable:
  - "When NOT to use this" (anti-patterns)
  - "Common mistakes" section
  - Decision matrix when there are multiple options
```

---

## 4. Version Control and Change Management

```
Versioning: semantic versioning (MAJOR.MINOR.PATCH)
  MAJOR: breaking change to existing guidance
  MINOR: new content added, backward-compatible
  PATCH: corrections, clarifications, minor updates

Version tracking:
  - Each document has version in frontmatter
  - CHANGELOG.md at repo root tracks significant changes
  - Git log is the authoritative history of all changes
```

### CHANGELOG format

```markdown
## [1.2.0] — 2026-06-27

### Added
- `13-AI/01-llm-engineering.md`: LLM engineering, RAG, Agents, Evals

### Changed
- `07-Security/03-authentication.md`: Updated password policy to align with NIST SP 800-63B rev4

### Fixed
- `09-Database/01-postgresql-patterns.md`: Corrected PgBouncer pool mode recommendation

### Deprecated
- `08-DevSecOps/01-cicd-security.md`: Trivy integration example — prefer Grype going forward
```

---

## 5. ADR Process

Architecture Decision Records (ADRs) document significant technical decisions.

### When to write an ADR

Write an ADR when:
- Choosing between multiple viable technical approaches (database, framework, architecture pattern)
- Deviating from handbook guidance (document why)
- Making a decision that will be hard or expensive to reverse
- Introducing a new significant dependency or technology

Do NOT write an ADR for:
- Implementation details within an already-decided approach
- Decisions that can easily be revisited and changed cheaply

### ADR template

See `adr/0000-template.md` for the full template.

```
Status options:
  proposed  → under discussion
  accepted  → decision made, in effect
  rejected  → considered but not adopted
  deprecated → was accepted, no longer recommended
  superseded → replaced by ADR-XXXX

File naming: adr/NNNN-short-title.md (zero-padded, sequential)
Examples:
  adr/0001-use-postgresql-over-mongodb.md
  adr/0002-adopt-kubernetes-for-all-services.md
  adr/0015-replace-rest-with-graphql-in-client-api.md
```

---

## 6. Review and Currency

### Review schedule

| Content type | Review cadence |
|---|---|
| Security standards | Quarterly |
| Compliance (GDPR/LGPD/PCI/ISO) | Annually or when regulation changes |
| Cloud patterns | Bi-annually |
| Technology-specific (Node.js, React) | With each major framework version |
| Checklists | Quarterly (update from incident learnings) |
| AI/LLM patterns | Monthly (field changes fast) |

### Staleness signals

An entry may be stale if:
- The version number in frontmatter is > 2 years old
- The code examples use APIs from deprecated library versions
- There have been major security advisories in the domain
- Engineers are consistently doing something different in practice

### Sunset process

To remove guidance:
1. Mark as `deprecated` in frontmatter with reason and date
2. Point to replacement (`See: ...`)
3. After 90 days, delete the file
4. Update CHANGELOG.md

---

## 7. Enforcement and Compliance

### Required vs recommended

```
MUST / SHALL:    Mandatory — deviations require documented exception
SHOULD:          Strongly recommended — deviations should be noted in PR
MAY / CAN:       Optional — engineering judgment call

Examples:
  MUST: SQL queries MUST use parameterized statements (no exceptions)
  SHOULD: Functions SHOULD be ≤ 50 lines (longer is justified sometimes)
  MAY: You MAY use either bcrypt or Argon2 for password hashing
```

### Enforcement mechanisms

- **CI/CD**: Automated checks (Semgrep rules, ESLint, type checking) enforce MUST items
- **Code review**: Reviewers expected to flag deviations from SHOULD items
- **Security review**: Required for new features touching auth/payments/PII (see `26-Checklists/#3-security-review`)
- **Architectural review**: Required for new services and significant architecture changes

### Exception process

When you have a valid reason to deviate from a MUST:

1. Document the exception in your PR description
2. Explain the reason (constraint, timeline, technical incompatibility)
3. Get approval from the relevant lead (security exception → security team)
4. Create a ticket to resolve the exception if it's temporary
5. For persistent exceptions: propose a handbook change so others benefit

_"We didn't have time"_ is not an acceptable exception for security MUST items. If a deadline conflicts with a security requirement, escalate — don't silently skip the requirement.
