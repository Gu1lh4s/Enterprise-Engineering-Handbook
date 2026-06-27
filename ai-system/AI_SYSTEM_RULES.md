# AI System Rules

> **Version:** 1.0.0
> **Status:** Active
> **Applies to:** All AI assistants operating within projects governed by this handbook

---

## Purpose

This document defines the mandatory operating rules for any AI system (Claude, GPT-4, Gemini, Copilot, or any LLM-based tool) when generating, reviewing, or modifying code, architecture, or technical documentation within projects that reference this handbook.

These rules exist because:

1. AI systems without constraints produce code that is functionally correct but architecturally naive, insecure by default, and compliance-blind
2. A consistent rule system eliminates the need to re-state requirements in every prompt
3. Engineering quality degrades when AI shortcuts replace deliberate design

---

## Rule Categories

| Category | ID Prefix | Description |
|---|---|---|
| Pre-generation | `PRE-*` | What the AI must do before writing any code |
| Code quality | `CQ-*` | Standards for code generated |
| Security | `SEC-*` | Security requirements for all output |
| Architecture | `ARC-*` | Architectural constraints |
| Compliance | `COM-*` | Legal and regulatory requirements |
| Output format | `OUT-*` | How responses must be structured |
| Forbidden | `DENY-*` | Actions the AI must never take |

---

## PRE — Pre-Generation Rules

### PRE-001: Read before writing

Before generating any code that touches an existing codebase, the AI must read:
- The file(s) being modified
- Direct dependencies of those files
- Any configuration or environment files relevant to the change

**Violation:** Generating code based on assumed file content leads to conflicts, type errors, and silent regressions.

### PRE-002: Understand the blast radius

Before any change, the AI must identify:
- Which systems or users are affected
- Whether the change is reversible
- What the failure mode is if the change is wrong

For changes with large blast radius (database migrations, auth changes, public API modifications), the AI must explicitly state the risk before proceeding.

### PRE-003: Security pre-check

Before generating code that handles any of the following, the AI must apply the Security Review System (`SECURITY_REVIEW_SYSTEM.md`):
- User authentication or authorization
- Data input from external sources (forms, APIs, webhooks)
- Database queries
- File uploads or downloads
- Payment data or PII
- Cryptographic operations
- Network requests to external services
- Environment variables or secrets

### PRE-004: Compliance pre-check

Before generating code that processes personal data, the AI must identify:
- Whether GDPR applies (EU users or EU-established company)
- Whether LGPD applies (Brazilian users or Brazil-established company)
- Whether PCI-DSS applies (payment card data)
- Whether HIPAA applies (health data, US context)

If any regulation applies, the AI must apply the relevant compliance module from `docs/18-LGPD/`, `docs/19-GDPR/`, or `docs/20-PCI/`.

### PRE-005: Identify the scope

The AI must not gold-plate. The scope of a change is exactly what was asked. Adding unrequested features, refactors, or abstractions is a violation. If a bug fix is requested, fix the bug. If the bug reveals a deeper problem, note it — do not fix it unasked.

---

## CQ — Code Quality Rules

### CQ-001: No dead code

Generated code must not include:
- Commented-out code blocks
- Unused imports, variables, or parameters
- Functions that are defined but never called
- Feature flags that will never be toggled

### CQ-002: Naming is documentation

Variable, function, class, and module names must be self-documenting. The AI must not generate names like `data`, `temp`, `result`, `obj`, `val`, `flag`, `item` without a domain qualifier. Use `userSession`, `paymentResult`, `invoiceItem`.

### CQ-003: No surprise side effects

Functions must do one thing. A function that fetches data and also writes to a database is a violation. Side effects must be explicit in the function signature (via parameters or return types, not hidden in the body).

### CQ-004: Error handling at boundaries only

The AI must not add `try/catch` or error handling to internal functions that cannot fail in normal operation. Error handling belongs at system boundaries:
- HTTP request handlers
- Database query execution
- External API calls
- File system operations
- User input processing

### CQ-005: Comments only for non-obvious WHY

The AI must not generate:
- Comments that restate what the code does (`// increment counter`)
- TODOs that reference the current task
- Docstrings that describe parameters when the parameter names are self-evident
- Multi-line comment blocks

The only valid comments explain a non-obvious constraint, a workaround for a known bug, or a subtle invariant.

### CQ-006: Immutability by default

In JavaScript/TypeScript, use `const` by default. In Python, prefer tuples over lists when mutation is not needed. In Go, prefer value types over pointer types when the struct is small and mutation is not needed. Mutation must be intentional and visible.

### CQ-007: No magic numbers or strings

All numeric constants and string literals that carry business meaning must be named:

```typescript
// VIOLATION
if (retryCount > 3) { ... }
if (status === 'pending') { ... }

// CORRECT
const MAX_RETRY_COUNT = 3
const BookingStatus = { PENDING: 'pending', CONFIRMED: 'confirmed' } as const
if (retryCount > MAX_RETRY_COUNT) { ... }
if (status === BookingStatus.PENDING) { ... }
```

### CQ-008: TypeScript strictness

TypeScript code must be generated with strict mode in mind:
- No implicit `any`
- No non-null assertions (`!`) unless the value is structurally guaranteed to be non-null
- No type casting (`as SomeType`) unless there is no alternative and the reason is documented
- Return types must be explicit on all exported functions

---

## SEC — Security Rules

### SEC-001: Never trust user input

All data from external sources (HTTP requests, query parameters, form submissions, file uploads, message queues, webhooks) is untrusted by default. The AI must apply:
- Type validation (is this actually a number/string/date?)
- Range validation (is this within acceptable bounds?)
- Format validation (does this match the expected pattern?)
- Sanitization before any use in HTML, SQL, shell commands, or file paths

### SEC-002: Parameterized queries only

The AI must never generate SQL with string interpolation or concatenation:

```typescript
// VIOLATION — SQL Injection vector
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`)

// CORRECT — Parameterized
const result = await db.query('SELECT * FROM users WHERE email = $1', [email])
```

This applies to all database libraries. ORMs that accept raw query strings must use their parameterized equivalents.

### SEC-003: Output encoding

Before inserting any user-supplied data into HTML, the AI must HTML-encode it. Never use `innerHTML` with user data. Use `textContent` for plain text, or a sanitization library (DOMPurify) when HTML rendering is required.

### SEC-004: Secrets never in code

The AI must never generate code that contains:
- API keys, tokens, or passwords hardcoded in source
- Connection strings with credentials
- Private keys or certificates
- Any secret in a comment

Secrets must always be read from environment variables. The AI must use `process.env.SECRET_NAME` (Node.js), `os.environ['SECRET_NAME']` (Python), or `Deno.env.get('SECRET_NAME')` (Deno) and include a runtime check:

```typescript
const apiKey = process.env.PAYMENT_API_KEY
if (!apiKey) throw new Error('PAYMENT_API_KEY is not configured')
```

### SEC-005: Authentication before authorization

The AI must never generate an authorization check without first verifying authentication. The pattern is:
1. Is there a valid session/token? (authentication)
2. Does this identity have permission? (authorization)
3. Does this identity own this resource? (ownership)

Skipping step 1 and jumping to step 2 is a broken access control vulnerability (OWASP A01:2021).

### SEC-006: Principle of least privilege

Generated code must request only the permissions it needs:
- Database connections must use users scoped to the required tables and operations
- API tokens must have the minimum required scopes
- IAM roles must follow least-privilege
- RLS (Row Level Security) policies must be table-specific

### SEC-007: No eval, no dynamic code execution

The AI must never generate:
- `eval()` in JavaScript/Python
- `exec()` in Python
- `Function()` constructor in JavaScript
- `system()` in any language
- Shell command construction from user input

### SEC-008: Cryptography is not DIY

The AI must never implement custom cryptographic algorithms. Use:
- For hashing passwords: `bcrypt`, `argon2id`, or `scrypt`
- For signing: `RS256`, `ES256` (JWT), or HMAC-SHA256
- For encryption: AES-256-GCM
- For key derivation: PBKDF2 or HKDF
- For random numbers: `crypto.randomBytes()` (Node.js), `secrets` module (Python), `crypto.getRandomValues()` (browser)

Never use `Math.random()` for anything security-related.

### SEC-009: HTTPS everywhere

Generated code must not reference `http://` endpoints for production traffic. All internal service-to-service calls must use TLS. The AI must flag any HTTP-only configuration as a security violation.

### SEC-010: Dependency awareness

When adding a new dependency, the AI must:
- Use the latest stable version (not `@latest` in production — pin to a specific version)
- Note if the package has known CVEs
- Prefer packages with >1M weekly downloads and active maintenance

---

## ARC — Architecture Rules

### ARC-001: Separation of concerns

The AI must not mix layers:
- HTTP handlers must not contain business logic
- Business logic must not contain database queries
- Database queries must not contain presentation formatting

### ARC-002: Explicit over implicit

Magic and convention-over-configuration are acceptable in well-understood frameworks (Rails, Django, Next.js). In custom code, behavior must be explicit. Hidden dependencies, global state, and implicit initialization are violations.

### ARC-003: Design for failure

Every external call (database, API, queue) must be treated as if it will fail:
- Define the retry policy
- Define the timeout
- Define the circuit breaker behavior
- Define what happens to in-flight data when the service is unavailable

### ARC-004: Idempotency for state-changing operations

APIs and functions that mutate state (payments, emails, database writes) must be idempotent. The AI must include an idempotency key pattern for any POST/PUT/DELETE operation that:
- Involves money
- Sends a notification
- Creates a unique resource

### ARC-005: No premature abstraction

The AI must not create abstractions that don't exist yet. Three lines of similar code is better than a premature helper. Abstract only when:
- The pattern has been repeated at least 3 times
- The abstraction is simpler than the instances it replaces
- The abstraction does not introduce hidden coupling

---

## COM — Compliance Rules

### COM-001: Data minimization

The AI must not generate code that collects more personal data than is strictly necessary for the stated purpose. If a feature requires only an email address, the schema must not include name, phone, or birth date unless explicitly required.

### COM-002: Explicit consent before data collection

Any feature that collects personal data must include a consent mechanism. The AI must flag data collection without consent as a compliance violation.

### COM-003: Data retention

Generated schemas and services that store personal data must include:
- A `created_at` timestamp
- A mechanism to delete or anonymize data (right to erasure — GDPR Art. 17, LGPD Art. 18)

### COM-004: PII in logs is forbidden

The AI must never generate logging statements that include:
- Email addresses
- Phone numbers
- Full names
- IP addresses (in GDPR contexts)
- Payment card numbers (always)
- Passwords or tokens (always)

Use identifiers (user IDs, request IDs) in logs. Never PII.

### COM-005: Audit trails for sensitive operations

Operations that touch financial data, authentication, or personal data changes must write to an immutable audit log including:
- Who performed the action (user ID)
- What action was taken
- When (UTC timestamp)
- What changed (before/after state)

---

## OUT — Output Format Rules

### OUT-001: State what you're doing before you do it

For any non-trivial operation, the AI must output one sentence describing what it is about to do before the first tool call or code block.

### OUT-002: State the risk for destructive operations

Before any operation that deletes data, modifies schemas, pushes to remote, or changes configuration: explicitly state what will be lost or changed and ask for confirmation.

### OUT-003: Summarize changes concisely

After completing a task, state in one or two sentences what changed and what the user should do next, if anything. Do not recap everything that was done line by line.

### OUT-004: Flag security findings inline

When reviewing or modifying code and a security issue is identified (even if outside the scope of the request), flag it explicitly with `[SECURITY]` prefix before continuing.

### OUT-005: Flag compliance findings inline

Same as OUT-004 but for GDPR, LGPD, PCI, or other regulatory issues. Use `[COMPLIANCE]` prefix.

---

## DENY — Forbidden Actions

### DENY-001: Never commit secrets

The AI must refuse to commit, push, or write to any file a value that appears to be a secret (matches patterns for API keys, tokens, private keys, or passwords).

### DENY-002: Never bypass security controls

The AI must never:
- Add `--no-verify` to git commands
- Disable linting or type checking to make a build pass
- Comment out security middleware to fix a bug
- Set `dangerouslySetInnerHTML` without sanitization
- Add `// @ts-ignore` or `// eslint-disable` without documenting the reason

### DENY-003: Never assume authorization

The AI must never generate code that assumes a user is authorized based on client-side state. Authorization must be validated server-side on every request.

### DENY-004: Never write to production directly

The AI must never generate code or scripts that:
- Directly modify production databases
- Force-push to main/master branches
- Delete production infrastructure
- Disable monitoring or alerting

### DENY-005: Never expose stack traces to users

Error responses to clients must never include:
- Stack traces
- Database query text
- Internal file paths
- Framework version information
- Environment variable names

---

## How to Reference This Document

When starting a new session with an AI assistant, include the following in your system prompt or initial message:

```
Read and follow the rules in this repository's ai-system/AI_SYSTEM_RULES.md before generating any code or making any decisions.
```

Or, when using Claude Code, add to your `CLAUDE.md`:

```markdown
## Engineering Standards
Follow all rules defined in ai-system/AI_SYSTEM_RULES.md.
Apply SECURITY_REVIEW_SYSTEM.md before touching any auth, data, or payment code.
Apply CODE_REVIEW_SYSTEM.md when reviewing pull requests.
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-06-27 | Initial release — PRE, CQ, SEC, ARC, COM, OUT, DENY categories |

---

## References

- [OWASP Top 10 2021](https://owasp.org/Top10/)
- [OWASP ASVS 4.0](https://owasp.org/www-project-application-security-verification-standard/)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [GDPR — Official Text](https://gdpr-info.eu/)
- [LGPD — Lei 13.709/2018](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm)
- [CWE Top 25 Most Dangerous Software Weaknesses](https://cwe.mitre.org/top25/)
- [Google Engineering Practices](https://google.github.io/eng-practices/)
- [Stripe Engineering Blog](https://stripe.com/blog/engineering)
