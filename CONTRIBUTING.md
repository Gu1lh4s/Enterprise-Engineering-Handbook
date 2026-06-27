# Contributing to the Enterprise Engineering Handbook

Thank you for considering a contribution. This handbook is written to the standard of Staff and Principal Engineers. Every PR is reviewed against that standard — not to be gatekeeping, but because a handbook that mixes high-quality and low-quality content loses its credibility as a reference.

---

## Quality Bar

A contribution is accepted if and only if it would be useful to a Staff Engineer at Stripe, Cloudflare, or a similar company. That means:

- **Depth over breadth** — A 10-page treatment of one topic is better than a 1-page overview of 10 topics
- **Correctness over completeness** — Do not add a section if you are not confident it is accurate
- **Examples over abstractions** — Show real code. Show real failure modes. Show what actually happens, not what the docs say happens
- **References to primary sources** — Link to RFCs, CVEs, official docs, and academic papers — not to Medium posts that summarize them

---

## What We Accept

- New sections that meet the quality bar and fit the structure
- Corrections to factual errors (with source citations)
- New examples, anti-patterns, or case studies
- New AI system rules (with rationale)
- Diagrams that clarify complex concepts

## What We Do Not Accept

- Surface-level content ("use prepared statements to prevent SQL injection" without depth)
- Content copied from other sources without citation
- Opinions presented as facts
- Content that duplicates an existing section
- Formatting changes without content improvement
- AI-generated content that has not been reviewed and verified by a human engineer

---

## Section Template

Every new document must use this frontmatter and structure:

```markdown
---
title: [Section Title]
category: [Category from docs/ structure]
version: 1.0.0
status: draft | review | stable
last_reviewed: YYYY-MM-DD
reviewers: [GitHub usernames]
references:
  - [Official source 1]
  - [Official source 2]
---

# [Title]

## Overview
## History and Context
## How It Works
## Types / Variants
## Examples — Vulnerable
## Examples — Correct
## Anti-Patterns
## Threat Model
## Best Practices
## Performance Considerations
## Compliance Mapping
## Checklist
## References
```

---

## Pull Request Process

1. Fork the repository
2. Create a branch: `docs/[section-name]` or `fix/[short-description]`
3. Write the content following the section template
4. Self-review against the quality bar checklist below
5. Open a PR with:
   - The section you added/modified
   - Your background (optional but appreciated — helps reviewers calibrate feedback)
   - Sources consulted

### Quality Bar Checklist (self-review before PR)

- [ ] The content is accurate and I can vouch for it
- [ ] At least one real-world example or case study is included
- [ ] At least one anti-pattern with root cause analysis is included
- [ ] A checklist or actionable summary exists
- [ ] References to primary sources (RFCs, CVEs, official docs) are included
- [ ] OWASP or NIST cross-reference included where applicable
- [ ] No PII, secrets, or proprietary information included

---

## ADR Process

Significant changes to the handbook structure or AI system rules require an Architecture Decision Record (ADR):

1. Copy the ADR template from `/adr/0000-template.md`
2. Number it sequentially (`0002-`, `0003-`, etc.)
3. Fill in Context, Decision, Options Considered, and Consequences
4. Open the PR — the ADR is part of the PR

---

## Review Timeline

PRs are reviewed within 7 days. If a PR has no response after 14 days, ping in the issues tab.
