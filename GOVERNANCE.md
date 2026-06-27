# Governance

## Versioning

This handbook uses [Semantic Versioning](https://semver.org/):

- **MAJOR** — Breaking structural changes, or a major standard update (e.g., new OWASP Top 10)
- **MINOR** — New sections or significant content additions
- **PATCH** — Corrections, clarifications, updated references

## Change Process

1. All changes go through pull requests — no direct commits to `main`
2. Every significant structural or AI system change requires an ADR
3. Breaking changes to AI system rules require a MAJOR version bump and migration notes

## Normative References

When a standard is referenced (OWASP, NIST, ISO, GDPR), always cite:
- The version or year of the standard
- The specific section or control
- The official URL

Standards evolve. A reference without a version becomes misleading over time.

## Content Review Cycle

Sections are reviewed annually for:
- Accuracy against current standards
- Relevance given technology changes
- Updated CVE or threat references
