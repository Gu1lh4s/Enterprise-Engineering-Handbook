# PCI-DSS — Payment Card Industry Data Security Standard

> **Category:** Compliance
> **Version:** 1.0.0
> **Standard:** PCI DSS v4.0 (March 2022, mandatory from March 2025)
> **Applicable to:** Anyone who stores, processes, or transmits cardholder data
> **Penalties:** Fines per incident, card brand termination, increased fees

---

## Table of Contents

1. [Overview and Scope](#1-overview-and-scope)
2. [Cardholder Data Environment (CDE)](#2-cardholder-data-environment-cde)
3. [12 PCI-DSS Requirements](#3-12-pci-dss-requirements)
4. [PCI-DSS v4.0 Key Changes](#4-pci-dss-v40-key-changes)
5. [SAQ — Self-Assessment Questionnaire](#5-saq--self-assessment-questionnaire)
6. [Scope Reduction Strategies](#6-scope-reduction-strategies)
7. [Compensating Controls](#7-compensating-controls)
8. [Technical Controls Reference](#8-technical-controls-reference)
9. [Stripe Integration Guide (SAQ A)](#9-stripe-integration-guide-saq-a)
10. [Incident Response for PCI](#10-incident-response-for-pci)
11. [Checklist](#11-checklist)
12. [References](#12-references)

---

## 1. Overview and Scope

PCI DSS is a contractual security standard mandated by the major card brands (Visa, Mastercard, American Express, Discover, UnionPay). It applies to any organization that:

```
- Stores cardholder data (PAN, CVV, expiry, cardholder name)
- Processes card transactions
- Transmits card data over networks

Even if you outsource payment processing entirely:
- Your website must be PCI-compliant (scoped as SAQ A or A-EP)
- Your infrastructure that touches cardholder data is in-scope
```

### Cardholder Data Types

| Data | Sensitive Authentication Data (SAD)? | Storable after auth? |
|---|---|---|
| Primary Account Number (PAN) | No | Yes (if encrypted + masked) |
| Cardholder Name | No | Yes |
| Expiration Date | No | Yes |
| Service Code | No | Yes |
| Full Magnetic Stripe | **Yes** | **NEVER** |
| CAV2/CVC2/CVV2/CID | **Yes** | **NEVER** |
| PIN / PIN Block | **Yes** | **NEVER** |

**Never store:**
- Full card magstripe data
- CVV/CVC (card verification value)
- PIN data

If you store PAN, it must be:
- Encrypted (AES-256 minimum)
- Masked (show only last 4 digits: ****-****-****-1234)
- Access controlled (only necessary staff/systems)
- Audited

---

## 2. Cardholder Data Environment (CDE)

The CDE is the set of people, processes, and technology that store, process, or transmit CHD/SAD, plus any systems that connect to or could affect their security.

```
In Scope:
- Web server (if redirects to payment form without iframe isolation)
- Application server processing card data
- Database storing PAN
- Network segments carrying card data
- Systems that manage/log payment processing
- Jump servers and bastion hosts that access CDE systems

Out of Scope (if properly isolated):
- Marketing website (no card data, separate network)
- HR systems
- Development environment (no real card data)

Reducing scope = reducing cost and complexity of compliance
```

---

## 3. 12 PCI-DSS Requirements

### Goal 1: Build and Maintain a Secure Network

**Requirement 1: Install and Maintain Network Security Controls**
```
- Firewall/NSG between internet and CDE
- Firewall/NSG between DMZ and internal network
- No direct public access to any database
- Deny all inbound/outbound traffic not explicitly required
- Stateful inspection enabled
- Document all traffic flows

Technical implementation:
- AWS Security Groups with explicit allow rules
- No 0.0.0.0/0 on ingress except ports 80/443 to web tier
- Database tier: only allow from application tier on specific port
- No default AWS VPC (default allows all internal traffic)
```

**Requirement 2: Apply Secure Configurations to All System Components**
```
- Change all vendor-supplied defaults (usernames, passwords, community strings)
- Remove unnecessary default accounts, services, protocols
- Enable only required services, ports, protocols
- Configure system security per industry-accepted hardening guides (CIS Benchmarks)

Technical implementation:
- CIS Benchmark hardening for AMIs / EC2 instances
- AWS Config rules for compliance checking
- Automated AMI baking with hardened baseline
- No default postgres/mysql passwords
```

### Goal 2: Protect Account Data

**Requirement 3: Protect Stored Account Data**
```
- Minimize data storage — delete cardholder data ASAP
- Do not store SAD after authorization
- Mask PAN when displayed (show only last 4 digits)
- Render PAN unreadable anywhere it is stored:
  - One-way hash (SHA-256 with salt)
  - Truncation (last 4 digits only)
  - Tokenization (store token, not PAN)
  - Strong encryption (AES-256 with proper key management)
- Key management: separate data encryption keys (DEK) from key encryption keys (KEK)
- Key rotation: rotate DEKs at least annually
```

**Requirement 4: Protect Cardholder Data with Strong Cryptography**
```
- TLS 1.2 or higher for all transmission
- No TLS 1.0 or 1.1
- No SSL
- Strong cipher suites only (no RC4, DES, 3DES)
- Valid certificates (not self-signed in production)
- HSTS header required
```

### Goal 3: Maintain a Vulnerability Management Program

**Requirement 5: Protect All Systems Against Malware**
```
- Antivirus/anti-malware on all applicable systems
- Antivirus must:
  - Detect all known types of malware
  - Be active and cannot be disabled by users
  - Generate audit logs
  - Keep definitions current
- Periodic malware evaluation for systems not commonly affected
```

**Requirement 6: Develop and Maintain Secure Systems and Software**
```
- Process for identifying and ranking vulnerabilities (CVSS scoring)
- Critical patches: within 1 month of release
- High patches: within 3 months
- Medium: within 6 months

Security in development:
- Secure development training for developers
- Code review for security issues before production
- SAST tools in CI/CD pipeline
- DAST testing of production-equivalent environments
- Web Application Firewall (WAF) in front of web-facing applications
- WAF must protect against OWASP Top 10 at minimum
- Change management for all production changes
```

### Goal 4: Implement Strong Access Control Measures

**Requirement 7: Restrict Access to System Components and Cardholder Data by Business Need to Know**
```
- Role-based access control
- Access only to what is needed for job function
- Default deny: if not explicitly allowed, it is denied
- Access rights reviewed at least every 6 months
```

**Requirement 8: Identify Users and Authenticate Access to System Components**
```
- Unique user IDs — no shared accounts
- Passwords: minimum 12 characters, complexity, history
- MFA required for all remote access to CDE
- MFA required for all access to web-based management interfaces
- Session timeout after 15 minutes of inactivity
- Lock account after max 10 failed attempts; minimum 30 minutes lockout
- Group accounts only for exceptional cases with documented justification
- Service accounts: rotate passwords, use secrets manager
```

**Requirement 9: Restrict Physical Access to Cardholder Data**
```
(Mainly applicable to on-premise data centers)
- Badged access to CDE areas
- CCTV monitoring with 90-day retention
- Visitor logs
- Media destruction procedures
- Tamper protection on POS devices
```

### Goal 5: Regularly Monitor and Test Networks

**Requirement 10: Log and Monitor All Access to System Components**
```
- Enable audit logs for:
  - Individual user access to cardholder data
  - Admin actions
  - Access to audit logs themselves
  - Invalid logical access attempts
  - Authentication (success and failure)
  - System-level object changes
  - Creation and deletion of system-level objects

Log content:
  - User identification
  - Type of event
  - Date/time
  - Success/failure
  - Origination of event
  - Identity of affected data/component

Log protection:
  - Logs must be protected from modification or deletion
  - Forward to central SIEM immediately
  - Retain for 12 months; 3 months immediately available
  - Review daily (automated correlation/alerting)
```

**Requirement 11: Test Security of Systems and Networks Regularly**
```
- Internal and external vulnerability scans quarterly (ASV for external)
- Penetration testing:
  - Network layer: at least annually + after significant change
  - Application layer: at least annually + after significant change
  - Segmentation testing: if segmentation used to reduce scope, verify annually
- Intrusion detection/prevention system (IDS/IPS)
- File integrity monitoring on CDE systems (detect unauthorized changes)
```

### Goal 6: Maintain an Information Security Policy

**Requirement 12: Support Information Security with Organizational Policies and Programs**
```
- Information security policy reviewed annually
- Annual security awareness training for all personnel
- Acceptable use policies for critical technologies
- Incident response plan documented, tested annually
- Risk assessment process (annual or significant change)
- Background screening for personnel in CDE roles
- Service provider management (monitor compliance)
- Responsibility matrix — who owns each requirement
```

---

## 4. PCI-DSS v4.0 Key Changes

PCI DSS v4.0 was released March 2022. PCI DSS v3.2.1 retired March 31, 2024. All new requirements must be met by March 31, 2025.

### Major Changes

```
1. Customized Approach (new)
   - Alternative to "defined approach" (traditional compliance)
   - Organizations can implement their own controls that meet the security objective
   - Requires documentation of how the control meets the objective

2. Multi-Factor Authentication (Req. 8.4/8.5)
   - v3.2.1: MFA required for remote access to CDE
   - v4.0: MFA required for ALL administrative access to CDE (even from internal network)
   - v4.0: MFA required for all users accessing web-based admin interfaces

3. Passwords (Req. 8.3.6)
   - v3.2.1: minimum 7 characters
   - v4.0: minimum 12 characters (or 8 if complexity requirements met)

4. Targeted Risk Analysis (Req. 12.3)
   - For any "best practice" requirements with flexible implementation timing
   - Organization must perform risk analysis to justify approach

5. Phishing Resistance
   - Security awareness training must cover phishing (Req. 12.6.3)
   - Phishing attacks at least every 6 months

6. WAF/Bot Detection (Req. 6.4.2)
   - v3.2.1: WAF "should" be deployed
   - v4.0: WAF or bot detection solution is required for all public-facing web apps

7. Sensitive Authentication Data in Scripts (Req. 6.4.3)
   - New: all payment page scripts must be authorized
   - Inventory of all scripts, authorization process, method to ensure integrity
```

---

## 5. SAQ — Self-Assessment Questionnaire

Most merchants don't need a full QSA audit. SAQ type depends on transaction type:

| SAQ | Who | Requirements |
|---|---|---|
| **SAQ A** | Card-not-present merchants using third-party payment page (iframe or redirect) | ~22 questions |
| **SAQ A-EP** | E-commerce with JavaScript payment library (Stripe.js, Braintree.js) | ~191 questions |
| **SAQ B** | Card-present merchants, no electronic data | Physical imprinters |
| **SAQ B-IP** | Card-present, IP terminals, no cardholder data stored | POS terminals |
| **SAQ C** | Payment apps on internet-connected systems | — |
| **SAQ C-VT** | Web-based virtual terminal | — |
| **SAQ D** | All others | 250+ questions (full compliance) |

### SAQ A Qualification

SAQ A = simplest. Requirements:
```
✓ All payment functions fully outsourced to a PCI-compliant third party
✓ Cardholder data NEVER touches your systems
✓ Your web server only renders HTML that loads the payment form via iframe from the third party
✓ Third party handles all card input, authorization, and token generation
✓ You store only the token and last 4 digits (if needed for display)

If you use Stripe Checkout or Stripe Elements with iframe = likely SAQ A
If you use Stripe.js (custom form) where script runs on your page = SAQ A-EP
```

---

## 6. Scope Reduction Strategies

### Tokenization

Replace PAN with a token. Token has no mathematical relationship to the PAN.

```typescript
// Your system only stores tokens, not PANs
// Stripe automatically tokenizes: your server receives payment_method_id, not card number

interface PaymentRecord {
  id: string
  userId: string
  stripeCustomerId: string        // cus_xxx
  stripePaymentMethodId: string   // pm_xxx (token for the card)
  cardBrand: 'visa' | 'mastercard' | 'amex'
  cardLast4: string               // "4242" — display only
  cardExpMonth: number
  cardExpYear: number
  createdAt: Date
}

// When charging:
const charge = await stripe.paymentIntents.create({
  amount: 5000,
  currency: 'brl',
  customer: user.stripeCustomerId,
  payment_method: user.stripePaymentMethodId,
  confirm: true,
})
// Stripe handles all PAN data — your system only uses the token
```

### Network Segmentation

```
Internet → WAF → Web Tier (DMZ)
                    ↓ (limited access)
           Application Tier (Private subnet)
                    ↓ (very restricted — only from app tier)
           Database Tier (No internet access)

If Database Tier has no path to internet:
- Even if app is compromised, attacker cannot exfiltrate from DB via internet
- Reduces CDE scope (application tier may not be in CDE if it only stores tokens)
```

---

## 7. Compensating Controls

When a required control cannot be implemented as specified (technical constraint, business limitation), a compensating control may be acceptable:

```
Requirements for a compensating control:
1. Meet the intent and rigor of the original requirement
2. Provide a similar level of defense as the original requirement
3. Address the risk that the original requirement mitigates
4. Must be "above and beyond" other requirements

Examples:
- Cannot implement MFA for legacy system: install IDS/IPS, additional logging, network isolation
- Cannot mask PAN in all storage: additional access controls + enhanced monitoring
```

---

## 8. Technical Controls Reference

### Database Security (Requirement 3)

```sql
-- Never store CVV/CVC in database
-- If must store PAN: use encryption

-- Proper PAN masking for display
CREATE OR REPLACE FUNCTION mask_pan(pan TEXT) RETURNS TEXT AS $$
BEGIN
  RETURN regexp_replace(pan, '(\d{4})\d+(\d{4})', '\1 **** **** \2');
END;
$$ LANGUAGE plpgsql;

SELECT mask_pan('4111111111111111');  -- Returns: 4111 **** **** 1111

-- Encryption for stored PAN (if you must store it)
-- Use pgcrypto with AES-256
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Store encrypted
INSERT INTO payment_methods (user_id, pan_encrypted)
VALUES ($1, pgp_sym_encrypt($2, current_setting('app.encryption_key')));

-- Retrieve and decrypt
SELECT pgp_sym_decrypt(pan_encrypted::bytea, current_setting('app.encryption_key'))
FROM payment_methods WHERE user_id = $1;

-- Column-level audit trigger
CREATE OR REPLACE FUNCTION audit_pan_access() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO pan_access_log (user_id, accessed_by, accessed_at, table_name)
  VALUES (NEW.user_id, current_user, NOW(), TG_TABLE_NAME);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Logging (Requirement 10)

```typescript
// PCI-compliant audit logging
interface PciAuditLog {
  timestamp: string          // ISO 8601 — from NTP-synchronized source
  userId: string             // Who
  eventType: string          // What
  success: boolean           // Result
  sourceIp: string           // Where
  resource: string           // Affected resource
  details?: string           // Additional info (never include PAN)
}

const PCI_AUDIT_EVENTS = [
  'CARD_ACCESS',             // Any access to cardholder data
  'CARD_CREATE',             // New payment method added
  'CARD_DELETE',             // Payment method removed
  'CHARGE_CREATED',          // Payment initiated
  'CHARGE_SUCCEEDED',
  'CHARGE_FAILED',
  'REFUND_ISSUED',
  'ADMIN_LOGIN',
  'ADMIN_CONFIG_CHANGE',
  'USER_PRIVILEGE_CHANGE',
  'AUDIT_LOG_EXPORT',        // Someone accessed audit logs
]

async function logPciEvent(event: PciAuditLog): Promise<void> {
  // Write to SIEM via syslog/HTTP (separate from application DB)
  // Logs must be append-only — no updates or deletes
  await siem.ingest({
    ...event,
    application: 'payment-service',
    environment: 'production',
    pci_scope: true,
  })
}
```

### WAF Configuration (Requirement 6.4.2)

```nginx
# ModSecurity + OWASP Core Rule Set (CRS)
# /etc/modsecurity/modsecurity.conf

SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On

# OWASP CRS
Include /etc/modsecurity/crs/crs-setup.conf
Include /etc/modsecurity/crs/rules/*.conf

# Custom rules for PCI
# Block attempts to access card data patterns
SecRule REQUEST_BODY "@rx \b4[0-9]{12}(?:[0-9]{3})?\b" \
  "id:10001,phase:2,block,msg:'Potential PAN in request body',tag:'PCI'"

# Block CVV patterns
SecRule REQUEST_BODY "@rx \b\d{3,4}\b" \
  "id:10002,phase:2,log,msg:'Potential CVV in request body',tag:'PCI'"
  # Note: This is a broad rule — tune based on your application
```

---

## 9. Stripe Integration Guide (SAQ A)

Using Stripe Checkout or Stripe Elements correctly results in SAQ A scope (simplest compliance).

```typescript
// Server-side: Create payment intent
// YOUR server creates the intent, but NEVER sees the card number
app.post('/api/payments/create-intent', authenticate, async (req, res) => {
  const { amount, currency } = req.body
  
  const paymentIntent = await stripe.paymentIntents.create({
    amount: Math.round(amount * 100),  // Convert to cents
    currency,
    customer: req.user.stripeCustomerId,
    metadata: {
      userId: req.user.id,
      orderId: req.body.orderId,
    },
    // Automatic payment methods
    automatic_payment_methods: { enabled: true },
  })
  
  // Only send the client_secret to the browser
  // client_secret allows completing the payment but not reading card data
  res.json({ clientSecret: paymentIntent.client_secret })
})

// Server-side: Store the result (webhook — most reliable)
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  const sig = req.headers['stripe-signature']!
  
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`)
  }
  
  if (event.type === 'payment_intent.succeeded') {
    const paymentIntent = event.data.object as Stripe.PaymentIntent
    
    await db.query(`
      UPDATE orders SET
        payment_status = 'paid',
        stripe_payment_intent_id = $1,
        paid_at = NOW()
      WHERE id = $2
    `, [paymentIntent.id, paymentIntent.metadata.orderId])
    
    // Log PCI audit event
    await logPciEvent({
      timestamp: new Date().toISOString(),
      userId: paymentIntent.metadata.userId,
      eventType: 'CHARGE_SUCCEEDED',
      success: true,
      sourceIp: 'stripe-webhook',
      resource: `payment_intent:${paymentIntent.id}`,
    })
  }
  
  res.json({ received: true })
})
```

```html
<!-- Client-side: Stripe Elements (card data NEVER touches your JavaScript) -->
<!-- Stripe renders the card input in a cross-origin iframe -->
<!DOCTYPE html>
<html>
<head>
  <!-- Load Stripe.js from Stripe's CDN — NOT self-hosted -->
  <script src="https://js.stripe.com/v3/"></script>
</head>
<body>
  <div id="payment-element"></div>
  <button id="submit">Pay</button>
  
  <script>
    const stripe = Stripe(PUBLISHABLE_KEY)  // pk_live_... or pk_test_...
    
    // Fetch clientSecret from your server
    const { clientSecret } = await fetch('/api/payments/create-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + token },
      body: JSON.stringify({ amount: 99.99, currency: 'brl' }),
    }).then(r => r.json())
    
    const elements = stripe.elements({ clientSecret })
    const paymentElement = elements.create('payment')
    paymentElement.mount('#payment-element')
    
    // User submits — Stripe collects card data in its iframe
    // Your JavaScript never sees the card number
    document.getElementById('submit').onclick = async () => {
      const { error } = await stripe.confirmPayment({
        elements,
        confirmParams: {
          return_url: 'https://yoursite.com/payment/complete',
        },
      })
      
      if (error) {
        // Show error to user (never log the error — may contain sensitive info)
        showError(error.message)
      }
    }
  </script>
</body>
</html>
```

---

## 10. Incident Response for PCI

If cardholder data is breached:

```
Immediate (0-24 hours):
1. Isolate compromised systems
2. Preserve evidence (forensic image before making changes)
3. Contact your payment processor/acquirer
4. Contact PCI forensic investigator (PFI) if required
5. Assess scope: how many cards, what time period, what data

Within 24 hours:
- Card brands may be notified by your acquirer
- Card brands will decide whether to reissue cards

Reporting:
- Acquirer: as soon as aware
- Card brands: via acquirer
- Law enforcement: as required by jurisdiction
- Data subjects: as required by GDPR/LGPD and breach notification laws

Post-incident:
- PCI Forensic Investigation (mandatory if significant breach)
- PFI determines root cause, affected systems, scope
- Implement remediation before cards reissued
- Verify remediation with QSA assessment
- Potential fines from card brands
- Potential increased transaction fees
```

---

## 11. Checklist

**Network Security**
- [ ] Firewall between internet and CDE (Req. 1)
- [ ] Firewall between DMZ and internal network (Req. 1)
- [ ] No default vendor passwords (Req. 2)
- [ ] Only necessary services/ports enabled (Req. 2)

**Data Protection**
- [ ] No SAD (CVV, magstripe, PIN) stored after authorization (Req. 3)
- [ ] PAN encrypted if stored (Req. 3)
- [ ] PAN masked in display (last 4 only) (Req. 3)
- [ ] TLS 1.2+ for all CHD transmission (Req. 4)
- [ ] No SSL/TLS 1.0/1.1 (Req. 4)

**Access Control**
- [ ] Unique user IDs (no shared accounts) (Req. 8)
- [ ] Passwords minimum 12 characters (Req. 8)
- [ ] MFA for all admin access to CDE (Req. 8)
- [ ] 15-minute session timeout (Req. 8)
- [ ] Account lockout after 10 failed attempts (Req. 8)
- [ ] Least privilege access (Req. 7)
- [ ] Access review every 6 months (Req. 7)

**Vulnerability Management**
- [ ] Critical patches within 1 month (Req. 6)
- [ ] WAF deployed in front of web applications (Req. 6)
- [ ] SAST/DAST in development pipeline (Req. 6)
- [ ] Antivirus on all applicable systems (Req. 5)

**Monitoring**
- [ ] Audit logs for all CDE access events (Req. 10)
- [ ] Logs retained 12 months (3 immediately available) (Req. 10)
- [ ] Daily log review / automated alerting (Req. 10)
- [ ] Quarterly vulnerability scans (Req. 11)
- [ ] Annual penetration testing (Req. 11)
- [ ] File integrity monitoring on CDE (Req. 11)

**Policy**
- [ ] Incident response plan documented and tested (Req. 12)
- [ ] Annual security awareness training (Req. 12)
- [ ] Annual risk assessment (Req. 12)
- [ ] Service provider PCI compliance verified annually (Req. 12)

---

## 12. References

- [PCI DSS v4.0 Standard Document](https://www.pcisecuritystandards.org/document_library/)
- [PCI SSC — Security Standards Council](https://www.pcisecuritystandards.org/)
- [SAQ Documents](https://www.pcisecuritystandards.org/document_library/#results)
- [Stripe PCI Compliance Guide](https://stripe.com/docs/security)
- [OWASP CRS (ModSecurity Rules)](https://coreruleset.org/)
- [PCI DSS Quick Reference Guide v4.0](https://www.pcisecuritystandards.org/pdfs/pci_ssc_quick_guide.pdf)
