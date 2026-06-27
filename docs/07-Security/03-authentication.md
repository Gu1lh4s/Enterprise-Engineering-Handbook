# Authentication & Authorization

> **Category:** Security
> **Version:** 1.0.0
> **OWASP:** A07:2021 — Identification and Authentication Failures
> **CWE:** CWE-287, CWE-306, CWE-522

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Password Authentication](#2-password-authentication)
3. [JSON Web Tokens (JWT)](#3-json-web-tokens-jwt)
4. [Session Management](#4-session-management)
5. [OAuth 2.0](#5-oauth-20)
6. [OpenID Connect (OIDC)](#6-openid-connect-oidc)
7. [Multi-Factor Authentication (MFA)](#7-multi-factor-authentication-mfa)
8. [API Keys and Service Authentication](#8-api-keys-and-service-authentication)
9. [Authorization — RBAC, ABAC, ReBAC](#9-authorization--rbac-abac-rebac)
10. [Common Vulnerabilities](#10-common-vulnerabilities)
11. [Secure Implementation Patterns](#11-secure-implementation-patterns)
12. [Checklist](#12-checklist)
13. [References](#13-references)

---

## 1. Fundamentals

### Authentication vs Authorization

| Concept | Question | Example |
|---|---|---|
| **Authentication (AuthN)** | Who are you? | Login with email+password |
| **Authorization (AuthZ)** | What are you allowed to do? | Only admins can delete users |
| **Identification** | Who do you claim to be? | The username/email provided |
| **Non-repudiation** | Can you prove you did it? | Cryptographic signatures on actions |

These are distinct and must be implemented separately. A system can authenticate correctly but authorize incorrectly (Insecure Direct Object References, horizontal privilege escalation).

### The Authentication Factors

| Factor | Type | Examples | Weakness |
|---|---|---|---|
| Something you know | Knowledge | Password, PIN, security question | Phishable, breachable |
| Something you have | Possession | Phone (SMS), hardware token, TOTP | Can be stolen, SIM-swapped |
| Something you are | Inherence | Fingerprint, face, iris | Cannot be changed if compromised |
| Somewhere you are | Location | IP range, GPS geofence | Spoofable |
| Something you do | Behavior | Typing rhythm, gait | Difficult to implement |

MFA combines ≥2 different factor types. Combining two knowledge factors (password + security question) is NOT MFA.

---

## 2. Password Authentication

### Storage: The Only Correct Approach

**Never store:**
- Plaintext passwords (SHA-1, SHA-256 are also unacceptable)
- Encrypted passwords (encryption is reversible)
- MD5, SHA-1 hashes (fast hashes — GPU can crack millions per second)

**Always use adaptive, slow hashing algorithms:**

```python
# Python — bcrypt (most common)
import bcrypt

def hash_password(password: str) -> str:
    salt = bcrypt.gensalt(rounds=12)  # 2^12 = 4096 iterations
    hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
    return hashed.decode('utf-8')

def verify_password(password: str, stored_hash: str) -> bool:
    return bcrypt.checkpw(password.encode('utf-8'), stored_hash.encode('utf-8'))

# Argon2 (winner of Password Hashing Competition 2015)
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(
    time_cost=3,       # iterations
    memory_cost=65536, # 64 MB
    parallelism=4,
    hash_len=32,
    salt_len=16
)

def hash_password(password: str) -> str:
    return ph.hash(password)

def verify_password(hash: str, password: str) -> bool:
    try:
        ph.verify(hash, password)
        if ph.check_needs_rehash(hash):
            # Rehash with new parameters and update in DB
            pass
        return True
    except VerifyMismatchError:
        return False
```

```typescript
// TypeScript — bcrypt
import bcrypt from 'bcrypt'

const SALT_ROUNDS = 12

async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS)
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash)
}
```

```go
// Go — bcrypt
import "golang.org/x/crypto/bcrypt"

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost) // cost=10
    return string(bytes), err
}

func VerifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### Hashing Algorithm Comparison

| Algorithm | Memory Hard? | Recommended? | Cost Factor |
|---|---|---|---|
| Argon2id | Yes | **Best choice** | time=3, mem=64MB |
| bcrypt | No | Good, widely available | cost=12 |
| scrypt | Yes | Good | N=32768, r=8, p=1 |
| PBKDF2-SHA256 | No | Acceptable (FIPS) | 600,000 iterations |
| MD5/SHA-1 | No | **Never** | — |
| SHA-256 (plain) | No | **Never** | — |

### Password Policy (NIST SP 800-63B)

**NIST 2024 recommendations (SP 800-63B-4):**
- Minimum 8 characters; allow up to 64+
- Check against known breached passwords (HaveIBeenPwned API)
- No arbitrary complexity rules (no "must have uppercase + symbol + number")
- No expiration-based rotation unless compromised
- No security questions
- Allow all printable ASCII characters + Unicode
- Do not truncate passwords (some old systems truncated at 8 chars)

```typescript
// Check against HaveIBeenPwned (k-anonymity — never sends full hash)
import crypto from 'crypto'

async function isPasswordBreached(password: string): Promise<boolean> {
  const hash = crypto.createHash('sha1').update(password).digest('hex').toUpperCase()
  const prefix = hash.slice(0, 5)
  const suffix = hash.slice(5)
  
  const response = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`)
  const text = await response.text()
  
  return text.split('\n').some(line => line.startsWith(suffix))
}

// If isPasswordBreached returns true: "This password has appeared in known data breaches. Please choose a different password."
```

### Login Rate Limiting and Brute Force Protection

```typescript
import rateLimit from 'express-rate-limit'
import { RedisStore } from 'rate-limit-redis'

// Per IP + per account rate limiting
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                    // 10 attempts per 15 min per IP
  store: new RedisStore({ /* ... */ }),
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many login attempts. Please try again in 15 minutes.',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
    })
  },
  standardHeaders: true,
  legacyHeaders: false,
})

// Per-account lockout (progressive)
async function trackFailedLogin(email: string): Promise<void> {
  const key = `login_failures:${email}`
  const failures = await redis.incr(key)
  await redis.expire(key, 3600)  // reset after 1 hour
  
  if (failures >= 5) {
    // Lock account temporarily
    await redis.setex(`account_locked:${email}`, 900, '1')  // 15-min lock
    await sendSecurityEmail(email, 'Account temporarily locked')
  }
}

// Constant-time response to prevent account enumeration
async function login(email: string, password: string) {
  const user = await db.findUserByEmail(email)
  
  if (!user) {
    // STILL hash the password — prevents timing attack to enumerate valid emails
    await bcrypt.compare(password, '$2b$12$dummyhashforfailedlookup')
    return { success: false, error: 'Invalid credentials' }
  }
  
  const valid = await bcrypt.compare(password, user.passwordHash)
  
  if (!valid) {
    await trackFailedLogin(email)
    return { success: false, error: 'Invalid credentials' }
  }
  
  return { success: true, user }
}
```

---

## 3. JSON Web Tokens (JWT)

### JWT Structure

A JWT is a Base64URL-encoded string in three parts separated by dots:

```
header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9    ← Header: {"alg":"HS256","typ":"JWT"}
.
eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoiYWRtaW5AZXhhbXBsZS5jb20iLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MzAwMDAwMDAsImV4cCI6MTczMDAwMzYwMH0
                                            ← Payload: {"sub":"user_123","email":"admin@example.com","role":"admin","iat":1730000000,"exp":1730003600}
.
HMACSHA256(header + "." + payload, secret)  ← Signature
```

**Standard claims:**
| Claim | Name | Description |
|---|---|---|
| `iss` | Issuer | Who issued the token |
| `sub` | Subject | Who the token represents (user ID) |
| `aud` | Audience | Intended recipient |
| `exp` | Expiration | Unix timestamp when token expires |
| `iat` | Issued at | Unix timestamp when token was issued |
| `nbf` | Not before | Token not valid before this time |
| `jti` | JWT ID | Unique identifier (for revocation) |

### Algorithms

```
HS256 / HS384 / HS512 — HMAC with SHA-256/384/512
  • Symmetric — same key signs and verifies
  • Simple but requires sharing secret between issuer and verifier
  • Not suitable when 3rd parties need to verify tokens

RS256 / RS384 / RS512 — RSA signature with SHA-256/384/512
  • Asymmetric — private key signs, public key verifies
  • Public key can be published (JWKS endpoint)
  • Widely supported, larger keys (2048+ bits)

ES256 / ES384 / ES512 — ECDSA with P-256/P-384/P-521
  • Asymmetric — smaller keys than RSA, same security
  • ES256 recommended for new implementations
  
EdDSA (Ed25519) — Edwards-curve Digital Signature Algorithm
  • Modern, fast, small signatures
  • Increasing adoption

NEVER USE: none algorithm (allows forging tokens)
NEVER USE: RS256 with HS256 confusion (alg confusion attacks)
```

### Critical Vulnerabilities

**1. Algorithm Confusion (alg:none)**
```javascript
// VULNERABLE — accepts "none" algorithm
const decoded = jwt.verify(token, secret, { algorithms: ['HS256', 'none'] })

// Attack: attacker crafts token with alg:none, removes signature
// Header: {"alg":"none","typ":"JWT"}
// Payload: {"sub":"admin","role":"admin","exp":9999999999}
// Token: eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiIsImV4cCI6OTk5OTk5OTk5OX0.

// SECURE — explicitly specify allowed algorithms
const decoded = jwt.verify(token, secret, { algorithms: ['HS256'] })
```

**2. RSA/HMAC Confusion Attack**
```javascript
// If server uses RS256 with public key, attacker may try:
// 1. Download the RSA public key (often publicly available)
// 2. Sign a token with HS256 using the PUBLIC key as the HMAC secret
// 3. Set alg:HS256 in the token header
// If the server accepts both alg types, it verifies HS256 with the public key = success

// SECURE — always explicitly specify the expected algorithm
const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256'] })
// Never allow both RS256 and HS256 on the same verification
```

**3. Header Injection via kid (Key ID)**
```javascript
// If kid is used to look up the signing key from a database
// VULNERABLE
const decoded = jwt.verify(token, keys[header.kid])
// If kid = "../../etc/passwd" or SQL injection

// SECURE — validate kid against allowlist
const VALID_KIDS = new Set(['key-2024-01', 'key-2024-06'])
if (!VALID_KIDS.has(header.kid)) throw new Error('Invalid key ID')
const key = await getKeyFromSecureStore(header.kid)
```

**4. Missing Expiry Validation**
```javascript
// Some libraries don't validate exp by default
// Always explicitly check exp
const decoded = jwt.verify(token, secret, {
  algorithms: ['HS256'],
  ignoreExpiration: false,  // default is false, but be explicit
  clockTolerance: 30,       // 30 seconds of leeway for clock skew
})
```

### Secure JWT Implementation

```typescript
import jwt from 'jsonwebtoken'
import crypto from 'crypto'

interface TokenPayload {
  sub: string      // user ID
  email: string
  role: 'user' | 'admin'
  jti: string      // unique token ID for revocation
}

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET!    // 256-bit random secret
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET!  // Different secret
const ACCESS_TOKEN_EXPIRY = '15m'    // Short-lived access token
const REFRESH_TOKEN_EXPIRY = '7d'    // Long-lived refresh token

function generateTokens(userId: string, email: string, role: 'user' | 'admin') {
  const jti = crypto.randomUUID()
  
  const accessToken = jwt.sign(
    { sub: userId, email, role, jti },
    ACCESS_TOKEN_SECRET,
    {
      algorithm: 'HS256',
      expiresIn: ACCESS_TOKEN_EXPIRY,
      issuer: 'app.example.com',
      audience: 'app.example.com',
    }
  )
  
  const refreshToken = jwt.sign(
    { sub: userId, jti: crypto.randomUUID() },
    REFRESH_TOKEN_SECRET,
    {
      algorithm: 'HS256',
      expiresIn: REFRESH_TOKEN_EXPIRY,
    }
  )
  
  // Store refresh token hash in DB (not the token itself)
  // Store jti for access token in revocation list when needed
  
  return { accessToken, refreshToken }
}

function verifyAccessToken(token: string): TokenPayload {
  const payload = jwt.verify(token, ACCESS_TOKEN_SECRET, {
    algorithms: ['HS256'],
    issuer: 'app.example.com',
    audience: 'app.example.com',
  }) as TokenPayload
  
  // Check if token has been revoked (by jti in Redis)
  // const revoked = await redis.get(`revoked_token:${payload.jti}`)
  // if (revoked) throw new Error('Token revoked')
  
  return payload
}
```

### Token Storage

| Storage | XSS Risk | CSRF Risk | Accessible from JS | Recommendation |
|---|---|---|---|---|
| `localStorage` | High | None | Yes | Avoid for auth tokens |
| `sessionStorage` | High | None | Yes | Better than localStorage |
| Cookies (no HttpOnly) | High | Yes | Yes | Avoid |
| Cookies (HttpOnly + SameSite=Strict) | None | None | No | **Best for access tokens** |
| Memory (JS variable) | Medium | None | Yes | Good for SPAs |

**Recommended architecture:**
```
Access Token  → HTTP-only cookie (short TTL, 15 min)
Refresh Token → HTTP-only cookie (long TTL, 7 days) + stored in DB
CSRF Token    → Custom header (X-CSRF-Token) or Double Submit Cookie pattern
```

### Token Rotation Pattern

```typescript
// Refresh token rotation — issue new refresh token on every use
// Old refresh token is invalidated (detect token reuse = breach indicator)
async function refreshAccessToken(refreshToken: string) {
  // 1. Verify refresh token
  const payload = jwt.verify(refreshToken, REFRESH_TOKEN_SECRET) as { sub: string; jti: string }
  
  // 2. Check if refresh token is in DB and hasn't been used
  const stored = await db.query(
    'SELECT * FROM refresh_tokens WHERE jti = $1 AND user_id = $2 AND revoked = false',
    [payload.jti, payload.sub]
  )
  
  if (!stored.rows.length) {
    // Token was already used — possible theft/reuse
    // Revoke ALL refresh tokens for this user
    await db.query('UPDATE refresh_tokens SET revoked = true WHERE user_id = $1', [payload.sub])
    throw new Error('Refresh token reuse detected — all sessions invalidated')
  }
  
  // 3. Revoke old refresh token
  await db.query('UPDATE refresh_tokens SET revoked = true WHERE jti = $1', [payload.jti])
  
  // 4. Issue new tokens
  const user = await db.findUserById(payload.sub)
  const tokens = generateTokens(user.id, user.email, user.role)
  
  // 5. Store new refresh token
  await db.query(
    'INSERT INTO refresh_tokens (jti, user_id, expires_at) VALUES ($1, $2, NOW() + INTERVAL \'7 days\')',
    [extractJti(tokens.refreshToken), user.id]
  )
  
  return tokens
}
```

---

## 4. Session Management

### Server-Side Sessions

Unlike JWTs, server-side sessions store state on the server. The client only holds a random session ID.

```typescript
import session from 'express-session'
import RedisStore from 'connect-redis'
import { createClient } from 'redis'
import crypto from 'crypto'

const redisClient = createClient({ url: process.env.REDIS_URL })

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET!,  // 256-bit random value
  name: '__Host-session',               // __Host- prefix prevents subdomain attacks
  resave: false,
  saveUninitialized: false,
  rolling: true,                        // Reset expiry on each request
  cookie: {
    httpOnly: true,
    secure: true,          // HTTPS only
    sameSite: 'strict',    // No cross-site requests
    maxAge: 30 * 60 * 1000,  // 30 minutes inactivity timeout
    path: '/',
  },
  genid: () => crypto.randomBytes(32).toString('hex'),  // Cryptographically random ID
}))

// Session fixation prevention — regenerate session ID on privilege change
app.post('/login', async (req, res) => {
  const user = await authenticate(req.body.email, req.body.password)
  if (!user) return res.status(401).json({ error: 'Invalid credentials' })
  
  // CRITICAL: Regenerate session ID after authentication (prevents session fixation)
  req.session.regenerate((err) => {
    if (err) return res.status(500).json({ error: 'Session error' })
    req.session.userId = user.id
    req.session.role = user.role
    res.json({ success: true })
  })
})
```

### Session vs JWT Comparison

| Property | Server Sessions | JWTs |
|---|---|---|
| State location | Server (Redis/DB) | Client (token) |
| Revocation | Instant (delete from store) | Requires blacklist |
| Scalability | Requires shared session store | Stateless — scales easily |
| Size | Small (session ID only) | Larger (payload in token) |
| Visibility | Opaque to client | Payload readable by client |
| Best for | Traditional web apps | Microservices, mobile APIs |

---

## 5. OAuth 2.0

### The Problem OAuth Solves

OAuth 2.0 allows a user to grant a third-party application limited access to their resources at another service — **without sharing their credentials**.

```
User → "Sign in with GitHub" → App (client) → Authorization Request → GitHub (authorization server)
GitHub authenticates user → User approves permissions → GitHub issues authorization code
Code sent to App redirect URI → App exchanges code for access token → App uses token to access GitHub API
```

### Grant Types

**Authorization Code (+ PKCE) — for web and mobile apps**
```
Best for: Web apps, mobile apps, SPAs
Tokens: Access token + optional refresh token
Security: High — code is short-lived, PKCE prevents interception
```

**Client Credentials — for server-to-server**
```
Best for: Backend services, daemons, microservices
No user involved — machine authentication
Tokens: Access token only
Security: Requires secure client_secret storage
```

**Device Authorization — for devices with limited input**
```
Best for: Smart TVs, CLI tools, IoT devices
User enters code on a separate device
```

**Implicit (deprecated) — avoid**
```
Was: for SPAs (simpler flow)
Problem: Access token in URL fragment — exposed in browser history, logs, Referer header
Solution: Use Authorization Code + PKCE instead
```

### Authorization Code Flow with PKCE

PKCE (Proof Key for Code Exchange — RFC 7636) prevents authorization code interception attacks.

```typescript
// 1. Generate PKCE code verifier and challenge
function generatePkce() {
  const verifier = crypto.randomBytes(32).toString('base64url')
  const challenge = crypto.createHash('sha256').update(verifier).digest('base64url')
  return { verifier, challenge }
}

// 2. Build authorization URL
function buildAuthUrl(provider: 'github' | 'google', redirectUri: string): string {
  const { verifier, challenge } = generatePkce()
  const state = crypto.randomBytes(16).toString('hex')
  
  // Store verifier and state in session (verify them on callback)
  session.oauthState = state
  session.pkceVerifier = verifier
  
  const params = new URLSearchParams({
    response_type: 'code',
    client_id: process.env.GITHUB_CLIENT_ID!,
    redirect_uri: redirectUri,
    scope: 'read:user user:email',
    state,                      // CSRF protection
    code_challenge: challenge,
    code_challenge_method: 'S256',
  })
  
  return `https://github.com/login/oauth/authorize?${params}`
}

// 3. Handle callback — exchange code for tokens
async function handleCallback(code: string, state: string, session: Session) {
  // Validate state (CSRF protection)
  if (state !== session.oauthState) throw new Error('Invalid state parameter')
  
  // Exchange code for tokens (with PKCE verifier)
  const response = await fetch('https://github.com/login/oauth/access_token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
    body: JSON.stringify({
      client_id: process.env.GITHUB_CLIENT_ID,
      client_secret: process.env.GITHUB_CLIENT_SECRET,
      code,
      redirect_uri: process.env.REDIRECT_URI,
      code_verifier: session.pkceVerifier,  // PKCE
    }),
  })
  
  const { access_token, refresh_token, error } = await response.json()
  if (error) throw new Error(`OAuth error: ${error}`)
  
  // Fetch user info with access token
  const userInfo = await fetchGitHubUser(access_token)
  
  // Create or update user in your DB
  const user = await upsertUser({ email: userInfo.email, provider: 'github', providerId: userInfo.id })
  
  return user
}
```

### Security Checklist for OAuth

- [ ] Always use PKCE for Authorization Code flow (even for confidential clients)
- [ ] Validate `state` parameter against session value (CSRF protection)
- [ ] Use short-lived authorization codes (GitHub codes expire after 10 minutes)
- [ ] Validate `redirect_uri` against registered values
- [ ] Store `client_secret` only in server-side environment variables (never in client)
- [ ] Validate `iss`, `aud`, `exp` in access tokens
- [ ] Use HTTPS for all OAuth endpoints
- [ ] Scope down to minimum required permissions

---

## 6. OpenID Connect (OIDC)

OIDC is built on top of OAuth 2.0 and adds **identity** (authentication) to OAuth's **authorization**. It introduces the **ID Token** — a JWT that identifies the authenticated user.

### OIDC vs OAuth

```
OAuth 2.0: Can user X access resource Y?  → Access token (opaque)
OIDC:      Who is user X?                 → ID token (JWT with user claims)
```

### OIDC Flow

```typescript
// OIDC Discovery — fetch endpoints from well-known configuration
async function getOidcConfig(issuer: string) {
  const response = await fetch(`${issuer}/.well-known/openid-configuration`)
  return response.json()
}

// Example config returned:
// {
//   "issuer": "https://accounts.google.com",
//   "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
//   "token_endpoint": "https://oauth2.googleapis.com/token",
//   "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
//   "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",  ← Public keys for ID token verification
//   "scopes_supported": ["openid","email","profile"],
//   "id_token_signing_alg_values_supported": ["RS256"]
// }

// Verify ID Token
import { createRemoteJWKSet, jwtVerify } from 'jose'

async function verifyIdToken(idToken: string, issuer: string, clientId: string) {
  const config = await getOidcConfig(issuer)
  const JWKS = createRemoteJWKSet(new URL(config.jwks_uri))
  
  const { payload } = await jwtVerify(idToken, JWKS, {
    issuer,
    audience: clientId,
    algorithms: ['RS256'],
  })
  
  // Validate nonce (anti-replay)
  if (payload.nonce !== session.oidcNonce) throw new Error('Invalid nonce')
  
  return payload  // Contains: sub, email, name, picture, email_verified, etc.
}
```

### OIDC Claims

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",           // Permanent, unique user identifier
  "aud": "812741506391.apps.googleusercontent.com",
  "iat": 1730000000,
  "exp": 1730003600,
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://lh3.googleusercontent.com/...",
  "locale": "en",
  "nonce": "abc123"                          // Anti-replay token
}
```

---

## 7. Multi-Factor Authentication (MFA)

### TOTP (Time-Based One-Time Passwords — RFC 6238)

The most common form of MFA for web applications. Apps: Google Authenticator, Authy, 1Password.

```typescript
import speakeasy from 'speakeasy'
import QRCode from 'qrcode'

// 1. Generate TOTP secret for user
async function setupMfa(userId: string, email: string) {
  const secret = speakeasy.generateSecret({
    name: `YourApp (${email})`,
    issuer: 'YourApp',
    length: 20,
  })
  
  // Store secret encrypted in DB (NOT plaintext)
  await db.query(
    'UPDATE users SET mfa_secret_encrypted = $1, mfa_enabled = false WHERE id = $2',
    [encrypt(secret.base32), userId]
  )
  
  // Generate QR code for authenticator app
  const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url!)
  
  return { qrCodeUrl, backupCodes: generateBackupCodes() }
}

// 2. Verify TOTP code during enrollment
async function verifyTotpAndEnable(userId: string, totpCode: string) {
  const user = await db.findUserById(userId)
  const secret = decrypt(user.mfa_secret_encrypted)
  
  const isValid = speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token: totpCode,
    window: 1,        // Allow 1 step (30s) before/after current time
  })
  
  if (!isValid) throw new Error('Invalid TOTP code')
  
  await db.query('UPDATE users SET mfa_enabled = true WHERE id = $1', [userId])
  return true
}

// 3. Verify during login
async function verifyMfaLogin(userId: string, totpCode: string): Promise<boolean> {
  const user = await db.findUserById(userId)
  if (!user.mfa_enabled) return true  // MFA not set up
  
  const secret = decrypt(user.mfa_secret_encrypted)
  
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token: totpCode.replace(/\s/g, ''),  // Strip spaces user might type
    window: 1,
  })
}
```

### Backup Codes

```typescript
// Generate one-time backup codes (used when phone is lost)
function generateBackupCodes(count = 10): string[] {
  return Array.from({ length: count }, () =>
    crypto.randomBytes(5).toString('hex').match(/.{1,4}/g)!.join('-')
  )
  // Produces codes like: "a3f7-9c2d-b1e8"
}

// Store hashed backup codes (not plaintext)
async function storeBackupCodes(userId: string, codes: string[]) {
  const hashes = await Promise.all(codes.map(code => bcrypt.hash(code, 10)))
  await db.query(
    'INSERT INTO backup_codes (user_id, code_hash, used) SELECT $1, unnest($2::text[]), false',
    [userId, hashes]
  )
}

// Use backup code
async function useBackupCode(userId: string, inputCode: string): Promise<boolean> {
  const codes = await db.query(
    'SELECT id, code_hash FROM backup_codes WHERE user_id = $1 AND used = false',
    [userId]
  )
  
  for (const row of codes.rows) {
    if (await bcrypt.compare(inputCode.replace(/-/g, ''), row.code_hash)) {
      await db.query('UPDATE backup_codes SET used = true, used_at = NOW() WHERE id = $1', [row.id])
      return true
    }
  }
  
  return false
}
```

### WebAuthn / Passkeys (FIDO2)

WebAuthn is phishing-resistant — the authenticator (hardware key, device biometric) proves possession by signing a challenge that includes the origin. Attackers cannot use stolen credentials on a different origin.

```typescript
import {
  generateAuthenticationOptions,
  generateRegistrationOptions,
  verifyAuthenticationResponse,
  verifyRegistrationResponse,
} from '@simplewebauthn/server'

const RP_ID = 'app.example.com'
const RP_NAME = 'My App'
const ORIGIN = 'https://app.example.com'

// Registration
async function beginRegistration(userId: string, email: string) {
  const userAuthenticators = await db.getAuthenticators(userId)
  
  const options = await generateRegistrationOptions({
    rpName: RP_NAME,
    rpID: RP_ID,
    userID: userId,
    userName: email,
    attestationType: 'none',
    excludeCredentials: userAuthenticators.map(a => ({
      id: a.credentialID,
      type: 'public-key',
    })),
    authenticatorSelection: {
      residentKey: 'preferred',
      userVerification: 'preferred',
    },
  })
  
  await redis.setex(`webauthn_challenge:${userId}`, 300, options.challenge)
  
  return options
}

async function finishRegistration(userId: string, response: any) {
  const expectedChallenge = await redis.get(`webauthn_challenge:${userId}`)
  
  const { verified, registrationInfo } = await verifyRegistrationResponse({
    response,
    expectedChallenge: expectedChallenge!,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID,
  })
  
  if (!verified || !registrationInfo) throw new Error('Verification failed')
  
  await db.saveAuthenticator(userId, {
    credentialID: registrationInfo.credentialID,
    credentialPublicKey: registrationInfo.credentialPublicKey,
    counter: registrationInfo.counter,
    deviceType: registrationInfo.credentialDeviceType,
  })
  
  await redis.del(`webauthn_challenge:${userId}`)
  return true
}
```

---

## 8. API Keys and Service Authentication

### API Key Design

```typescript
// API key format: prefix_randomBytes_checksum
// prefix: identifies the service (sk_ for secret, pk_ for public)
// This makes keys easy to detect in code/logs scanners

function generateApiKey(): { key: string; hash: string } {
  const prefix = 'sk'
  const random = crypto.randomBytes(24).toString('base64url')
  const key = `${prefix}_${random}`
  
  // Store HASH, not the key (treat like a password)
  const hash = crypto.createHash('sha256').update(key).digest('hex')
  
  return { key, hash }
}

// Show key ONCE at creation — user cannot retrieve it again
// Store only the hash in the database

async function createApiKey(userId: string, name: string, scopes: string[]) {
  const { key, hash } = generateApiKey()
  
  await db.query(
    'INSERT INTO api_keys (user_id, name, key_hash, scopes, created_at) VALUES ($1, $2, $3, $4, NOW())',
    [userId, name, hash, JSON.stringify(scopes)]
  )
  
  return key  // Return ONCE — never store plaintext
}

// Verify API key on request
async function verifyApiKey(key: string) {
  if (!key.startsWith('sk_')) throw new Error('Invalid API key format')
  
  const hash = crypto.createHash('sha256').update(key).digest('hex')
  
  const result = await db.query(
    'SELECT * FROM api_keys WHERE key_hash = $1 AND revoked = false AND (expires_at IS NULL OR expires_at > NOW())',
    [hash]
  )
  
  if (!result.rows.length) throw new Error('Invalid or expired API key')
  
  // Update last_used_at
  await db.query('UPDATE api_keys SET last_used_at = NOW() WHERE key_hash = $1', [hash])
  
  return result.rows[0]
}
```

### mTLS (Mutual TLS) for Service-to-Service

```nginx
# Nginx config for mTLS
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/server.crt;
    ssl_certificate_key /etc/ssl/server.key;
    
    # Require client certificate
    ssl_client_certificate /etc/ssl/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;
    
    location /api/ {
        # Client cert subject passed as header
        proxy_set_header X-Client-CN $ssl_client_s_dn_cn;
        proxy_pass http://backend;
    }
}
```

---

## 9. Authorization — RBAC, ABAC, ReBAC

### Role-Based Access Control (RBAC)

```typescript
// User has roles; roles have permissions
// users ←→ user_roles ←→ roles ←→ role_permissions ←→ permissions

type Permission = 
  | 'bookings:read' | 'bookings:write' | 'bookings:delete'
  | 'users:read' | 'users:write' | 'users:delete'
  | 'reports:read'

const ROLE_PERMISSIONS: Record<string, Permission[]> = {
  admin:   ['bookings:read', 'bookings:write', 'bookings:delete', 'users:read', 'users:write', 'users:delete', 'reports:read'],
  manager: ['bookings:read', 'bookings:write', 'users:read', 'reports:read'],
  staff:   ['bookings:read', 'bookings:write'],
  client:  ['bookings:read'],
}

async function hasPermission(userId: string, permission: Permission): Promise<boolean> {
  const user = await db.query(
    'SELECT r.name as role FROM user_roles ur JOIN roles r ON ur.role_id = r.id WHERE ur.user_id = $1',
    [userId]
  )
  
  const permissions = user.rows.flatMap(row => ROLE_PERMISSIONS[row.role] ?? [])
  return permissions.includes(permission)
}

// Express middleware
function requirePermission(permission: Permission) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user?.id
    if (!userId) return res.status(401).json({ error: 'Unauthorized' })
    
    const allowed = await hasPermission(userId, permission)
    if (!allowed) return res.status(403).json({ error: 'Forbidden' })
    
    next()
  }
}

// Usage
app.delete('/users/:id', requirePermission('users:delete'), async (req, res) => {
  // ...
})
```

### Attribute-Based Access Control (ABAC)

ABAC evaluates policies based on attributes of the user, resource, and environment.

```typescript
interface Policy {
  subject: Record<string, unknown>   // User attributes
  resource: Record<string, unknown>  // Resource attributes
  action: string
  environment: Record<string, unknown>  // Context (time, IP, etc.)
}

function evaluatePolicy(policy: Policy): boolean {
  // Rule: Users can only edit their own bookings
  // AND only during business hours
  // AND only if the booking is in the future
  
  const { subject, resource, action, environment } = policy
  
  if (action === 'booking:edit') {
    const isOwner = resource.ownerId === subject.userId
    const isBusinessHours = environment.hour >= 8 && environment.hour < 18
    const isFuture = resource.date > environment.now
    return isOwner && isBusinessHours && isFuture
  }
  
  // Rule: Admins can edit any booking, any time
  if (subject.role === 'admin' && action === 'booking:edit') return true
  
  return false
}
```

### Row-Level Security (PostgreSQL)

Row Level Security is authorization enforced at the database layer — no application code can bypass it.

```sql
-- Enable RLS on table
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;
ALTER TABLE bookings FORCE ROW LEVEL SECURITY;

-- Policy: Users can only see their own bookings
CREATE POLICY user_sees_own_bookings ON bookings
  FOR SELECT
  USING (client_id = auth.uid());  -- auth.uid() is the authenticated user's ID

-- Policy: Users can only update their own future bookings
CREATE POLICY user_updates_own_bookings ON bookings
  FOR UPDATE
  USING (
    client_id = auth.uid()
    AND scheduled_at > NOW()
    AND status NOT IN ('cancelled', 'completed')
  );

-- Policy: Admins can see everything
CREATE POLICY admin_sees_all ON bookings
  FOR ALL
  USING (auth.role() = 'admin');
```

---

## 10. Common Vulnerabilities

### Insecure Direct Object Reference (IDOR)

```typescript
// VULNERABLE — user can access any booking by changing the ID in the URL
app.get('/api/bookings/:id', authenticate, async (req, res) => {
  const booking = await db.query('SELECT * FROM bookings WHERE id = $1', [req.params.id])
  res.json(booking.rows[0])  // Returns even if it's another user's booking
})

// SECURE — always filter by the authenticated user's ID
app.get('/api/bookings/:id', authenticate, async (req, res) => {
  const booking = await db.query(
    'SELECT * FROM bookings WHERE id = $1 AND client_id = $2',  // Also filter by owner
    [req.params.id, req.user.id]
  )
  if (!booking.rows.length) return res.status(404).json({ error: 'Not found' })
  res.json(booking.rows[0])
})
```

### Mass Assignment

```typescript
// VULNERABLE — user can set any field including is_admin
app.post('/api/users', async (req, res) => {
  const user = await db.createUser(req.body)  // req.body.is_admin = true is accepted
  res.json(user)
})

// SECURE — explicit allowlist of settable fields
app.post('/api/users', async (req, res) => {
  const { name, email, phone } = req.body  // Destructure only what's allowed
  const user = await db.createUser({ name, email, phone })
  res.json(user)
})
```

### Broken Object Level Authorization (BOLA / OWASP API Security #1)

```typescript
// VULNERABLE — trusts user_id from request body
app.delete('/api/posts/:id', authenticate, async (req, res) => {
  const { userId } = req.body  // NEVER trust user-supplied userId
  await db.deletePost(req.params.id)
})

// SECURE — trust only the authenticated user's ID
app.delete('/api/posts/:id', authenticate, async (req, res) => {
  const deleted = await db.query(
    'DELETE FROM posts WHERE id = $1 AND author_id = $2 RETURNING id',
    [req.params.id, req.user.id]  // req.user.id from verified JWT/session
  )
  if (!deleted.rowCount) return res.status(403).json({ error: 'Forbidden' })
  res.json({ success: true })
})
```

---

## 11. Secure Implementation Patterns

### Defense in Depth Auth Middleware

```typescript
// Express middleware chain for authenticated, authorized routes
import { expressjwt as jwt } from 'express-jwt'

// 1. Rate limiting
const authRateLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 })

// 2. JWT verification
const verifyJwt = jwt({
  secret: process.env.ACCESS_TOKEN_SECRET!,
  algorithms: ['HS256'],
  requestProperty: 'user',
})

// 3. Active session check (token not revoked, user not locked)
const checkActiveUser = async (req: Request, res: Response, next: NextFunction) => {
  const userId = req.user?.sub
  const user = await db.findUserById(userId)
  
  if (!user || user.locked || user.deleted_at) {
    return res.status(401).json({ error: 'Account not active' })
  }
  
  next()
}

// Stack the middleware
router.use(authRateLimiter, verifyJwt, checkActiveUser)
```

### Security Headers for Auth Endpoints

```typescript
// Specific headers for auth endpoints
app.post('/api/auth/login', (req, res, next) => {
  // Prevent caching of auth responses
  res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate')
  res.setHeader('Pragma', 'no-cache')
  
  // Clear any existing auth cookies before login
  res.clearCookie('session')
  
  next()
})
```

---

## 12. Checklist

**Password Authentication**
- [ ] Passwords hashed with bcrypt (cost≥12), Argon2id, or scrypt
- [ ] Breached password check on registration and password change
- [ ] Minimum 8 characters, no forced expiry unless compromised
- [ ] Login rate limiting (per IP and per account)
- [ ] Constant-time comparison to prevent timing attacks
- [ ] Constant-time response to prevent account enumeration

**JWT**
- [ ] Short-lived access tokens (≤15 minutes)
- [ ] Explicit algorithm restriction (no `none`, no multiple alg)
- [ ] Validate `iss`, `aud`, `exp`, `nbf`
- [ ] Tokens stored in HttpOnly + Secure + SameSite=Strict cookies
- [ ] Refresh token rotation with reuse detection
- [ ] Token revocation mechanism (jti allowlist/blocklist)

**Sessions**
- [ ] Session ID regenerated after login (session fixation prevention)
- [ ] HttpOnly + Secure + SameSite=Strict cookies
- [ ] Short timeout (30 minutes inactivity)
- [ ] Server-side session store (Redis) for invalidation
- [ ] `__Host-` prefix on session cookie name

**OAuth / OIDC**
- [ ] PKCE for all Authorization Code flows
- [ ] State parameter validated against session (CSRF)
- [ ] Nonce validated in OIDC ID tokens (anti-replay)
- [ ] `redirect_uri` validated against registered allowlist
- [ ] ID token signature verified against JWKS endpoint
- [ ] `client_secret` never exposed to browser

**Authorization**
- [ ] Every endpoint checks both authentication AND authorization
- [ ] IDOR prevented: all DB queries filter by authenticated user's ID
- [ ] No mass assignment (explicit field allowlists)
- [ ] Admin functions require explicit admin role check
- [ ] Row Level Security enabled in PostgreSQL

**MFA**
- [ ] TOTP available (Google Authenticator compatible)
- [ ] Backup codes generated and shown once
- [ ] Backup codes stored as bcrypt hashes
- [ ] WebAuthn / Passkeys offered as option

---

## 13. References

- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP JWT Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [RFC 6749 — OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 7636 — PKCE](https://www.rfc-editor.org/rfc/rfc7636)
- [OpenID Connect Specification](https://openid.net/specs/openid-connect-core-1_0.html)
- [WebAuthn Specification](https://www.w3.org/TR/webauthn-3/)
- [PortSwigger Auth Labs](https://portswigger.net/web-security/authentication)
- [HaveIBeenPwned API](https://haveibeenpwned.com/API/v3)
