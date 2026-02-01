---
name: hipaa-compliance
description: Use when handling health data, patient records, medical systems, PHI, ePHI, healthcare APIs, EHR integrations, telehealth, health apps, or any code processing Protected Health Information — HIPAA Security Rule technical safeguards, 18 identifiers, Safe Harbor, Expert Determination, 164.312, access control, audit log, encryption at rest, encryption in transit, BAA, Business Associate Agreement, minimum necessary, breach notification, HHS, OCR, de-identification
---

# HIPAA Compliance Coding Guidelines

## 1. Overview

HIPAA (Health Insurance Portability and Accountability Act) applies to **Covered Entities** (healthcare providers, health plans, healthcare clearinghouses) and **Business Associates** (any entity that creates, receives, maintains, or transmits PHI on behalf of a Covered Entity). Three rules: the **Privacy Rule** (who can access PHI), the **Security Rule** (technical/admin/physical safeguards for ePHI), and the **Breach Notification Rule** (requirements when unsecured PHI is compromised). Penalties: $137-$68,928 per violation, annual caps of $2,067,813 per violation category, criminal penalties up to $250,000 and 10 years imprisonment.

## 2. What is PHI (Protected Health Information)?

PHI is **individually identifiable health information** relating to health condition, healthcare provision, or payment that identifies or could identify the individual. **ePHI** = PHI in electronic form (databases, APIs, files, logs, backups) — the developer's primary concern.

**Critical distinction:** Health data alone is NOT PHI. A blood pressure of 140/90 is not PHI. A blood pressure of 140/90 linked to patient John Smith, DOB 1985-03-15, IS PHI. PHI exists only when health data is linked to identifiers.

### The 18 HIPAA Identifiers

| # | Identifier | Examples |
|---|-----------|----------|
| 1 | Names | First, last, maiden, aliases |
| 2 | Geographic data smaller than state | Street address, city, ZIP code, county |
| 3 | Dates (except year) related to individual | Birth, admission, discharge, death dates; ages over 89 |
| 4 | Phone numbers | Home, mobile, work |
| 5 | Fax numbers | Any fax linked to individual |
| 6 | Email addresses | Personal or work email |
| 7 | Social Security Numbers | Full or partial SSN |
| 8 | Medical record numbers | MRN, chart numbers |
| 9 | Health plan beneficiary numbers | Insurance member IDs |
| 10 | Account numbers | Billing, financial accounts |
| 11 | Certificate/license numbers | Driver's license, professional licenses |
| 12 | Vehicle identifiers/serial numbers | VIN, license plates |
| 13 | Device identifiers/serial numbers | Medical device UDIs, implant serials |
| 14 | Web URLs | Patient portal URLs with identifiers |
| 15 | IP addresses | Client IPs associated with patient activity |
| 16 | Biometric identifiers | Fingerprints, voiceprints, retinal scans |
| 17 | Full-face photographs | Photos, medical imaging with facial features |
| 18 | Any other unique identifying number | Custom patient IDs, research subject numbers |

## 3. The Three Rules (Developer View)

| Rule | CFR Reference | What Developers Must Do |
|------|--------------|------------------------|
| **Privacy Rule** | 45 CFR 164 Subpart E | Enforce minimum necessary access. Build consent management. Role-based data filtering. Patient right-of-access APIs. |
| **Security Rule** | 45 CFR 164 Subpart C | Technical safeguards: encryption, access controls, audit logging, integrity checks, transmission security. |
| **Breach Notification** | 45 CFR 164.400-414 | Breach detection, audit trails for forensics, notification workflows. 60-day window. 500+ records = HHS + media. |

## 4. Technical Safeguards (45 CFR 164.312) — The Developer Core

> Full code examples: [resources/technical-safeguards.md](resources/technical-safeguards.md)

| Safeguard | CFR | Status | Description |
|-----------|-----|--------|-------------|
| Unique User Identification | 164.312(a)(2)(i) | REQUIRED | No shared accounts; every action tied to a unique user identity |
| Emergency Access Procedure | 164.312(a)(2)(ii) | REQUIRED | "Break glass" mechanism with full audit trail and compliance notification |
| Automatic Logoff | 164.312(a)(2)(iii) | ADDRESSABLE | Session timeout for unattended workstations (15 min recommended) |
| Encryption and Decryption | 164.312(a)(2)(iv) | ADDRESSABLE | AES-256-GCM for ePHI at rest; keys in HSM or KMS |
| Audit Controls | 164.312(b) | REQUIRED | Log ALL ePHI access (who, what, when, where, action, outcome); retain 6+ years; NEVER log PHI content |
| Integrity Controls | 164.312(c)(1) | REQUIRED | HMAC checksums, version tracking; detect unauthorized alteration of ePHI |
| Person/Entity Authentication | 164.312(d) | REQUIRED | MFA strongly recommended; short-lived tokens (30 min); signed system-to-system requests |
| Transmission Security | 164.312(e)(1) | ADDRESSABLE | TLS 1.2+ mandatory; never transmit PHI in URLs or query parameters |

## 5. Administrative Safeguards for Developers (45 CFR 164.308)

> Full code examples: [resources/technical-safeguards.md](resources/technical-safeguards.md)

- **Risk Analysis (164.308(a)(1))** — Map every location where ePHI lives: database tables, file storage, cache layers, message queues, log files, backups, third-party services, temporary files.
- **Access Management (164.308(a)(4))** — RBAC with minimum necessary. Define granular per-role permission maps (e.g., receptionist: read:demographics only; physician: full clinical access; billing: billing fields only; researcher: de-identified only).
- **Incident Response (164.308(a)(6))** — Anomaly detection: alert when a user's access count exceeds 3x their baseline in a 1-hour window.
- **Contingency Plan (164.308(a)(7))** — Encrypted backups with SHA-256 checksum verification. Log all backup operations. Test restore procedures regularly.

## 6. De-identification (Safe Harbor Method)

> Full code examples: [resources/deidentification.md](resources/deidentification.md)

Remove all 18 identifiers per 45 CFR 164.514(b). De-identified data is NOT PHI. Key techniques:

- **SQL views** that replace identifiers with research IDs (MD5 hash with separate salt), truncate ZIP to 3 digits (000 if population < 20K), keep only year from dates, cap age at 90
- **PHI detection patterns** — regex scanning for SSN, phone, email, MRN, dates, IPs before logging or exporting
- **Date shifting** — deterministic per-patient offset (-365 to +365 days) preserving relative intervals
- **Complete de-identification** — strip all 18 identifiers, keep clinical data (diagnoses, procedures, labs, medications)

## 7. Minimum Necessary Rule

> Full code examples: [resources/deidentification.md](resources/deidentification.md)

Query only needed fields. No `SELECT *` on PHI tables. Return only fields required for the user's role.

- **Billing staff** see only: id, insurance_provider, policy_number, billing_code
- **Nurses** see only: id, name, dob, vitals (blood_pressure, heart_rate, temperature)
- **Physicians** see: id, name, dob, vitals, diagnoses, lab results
- **Receptionists** see only: id, name, phone, appointment date

Filter responses server-side based on role before returning to client. Use ORM scopes to enforce column restrictions at the query level.

## 8. BAA (Business Associate Agreement) Requirements

A BAA is required whenever a third party creates, receives, maintains, or transmits PHI on your behalf. Using a service without a BAA to process PHI is a violation regardless of breach.

| Service | HIPAA-Eligible | Notes |
|---------|---------------|-------|
| AWS | Yes | Must enable BAA, eligible services only |
| Google Cloud | Yes | Must sign BAA, eligible services only |
| Azure | Yes | Must sign BAA |
| Firebase (standard) | No | Not HIPAA-eligible |
| Google Analytics | No | Cannot process PHI |
| Sentry (standard) | No | PHI may leak into error reports |
| Datadog | Yes (Enterprise) | Requires HIPAA add-on |
| Twilio / SendGrid | Yes | BAA available |
| Stripe | Yes | BAA for health-related payments |
| MongoDB Atlas | Yes | Dedicated tier with BAA |
| Vercel / Netlify | No | Not HIPAA-eligible |
| Supabase | Yes | Pro plan with BAA |
| Auth0 | Yes | Enterprise plan with BAA |

**Important:** Scrub PHI from error tracking payloads before sending to any service. Redact request data, strip local variables from stack frames.

## 9. Mobile and API Considerations

- **No PHI in push notifications** — visible on lock screen. Use generic messages ("You have a new message from your care team") and require auth before displaying PHI content.
- **No PHI in client-side storage without encryption** — never use localStorage/sessionStorage for PHI. Store reference IDs only; fetch PHI on demand with auth.
- **No PHI in URLs** — use POST with body. PHI in URLs leaks to browser history, server logs, Referer headers.
- **Rate limiting on PHI endpoints** to detect bulk extraction (e.g., 60 reads/min, 5 exports/hour).

## 10. Breach Response Coding Patterns

> Full code examples: [resources/breach-response.md](resources/breach-response.md)

- **Detection** — Monitor for: bulk access (>100 records), off-hours access, new IP addresses. Log all breach indicators. Auto-alert security team on high-severity indicators.
- **Containment** — Immediately: revoke all sessions, revoke all tokens, lock user account, snapshot audit logs. Log all containment actions.
- **Breach Log** — Track: detection time, detector, breach type, severity, affected record count/type, containment actions, notification status (HHS within 60 days, individuals, media if 500+ records), resolution.

## 11. Common Mistakes

| # | Mistake | Why It Violates HIPAA | Fix |
|---|---------|----------------------|-----|
| 1 | Logging PHI content | Logs are less protected than DBs; violates 164.312(b) | Log resource IDs and actions only |
| 2 | PHI in URLs/query params | Leaks to browser history, server logs, Referer headers | Use POST with encrypted body |
| 3 | `SELECT *` on PHI tables | Violates minimum necessary rule | Select only columns needed for role |
| 4 | Shared service accounts | No accountability; violates 164.312(a)(2)(i) | Unique identity per user/service |
| 5 | No audit trail | Directly violates 164.312(b) | Log every ePHI read/write/delete/export |
| 6 | PHI in error messages | May display to users or send to error tracking | Generic errors; scrub PHI from payloads |
| 7 | Unencrypted backups | Unsecured PHI; violates 164.312(a)(2)(iv) | Encrypt all backups at rest |
| 8 | PHI in push notifications | Visible on lock screen to anyone | Generic notifications; auth before PHI display |
| 9 | Services without BAA | Violation regardless of breach | Verify BAA with every third-party service |
| 10 | No session timeout | Unattended workstation risk; violates 164.312(a)(2)(iii) | 15-minute idle timeout |
| 11 | PHI in analytics events | Analytics platforms lack BAAs, store indefinitely | Opaque IDs only in analytics |
| 12 | PHI in client storage unencrypted | localStorage/cookies accessible to JS | Encrypt or fetch on demand |
| 13 | No de-identification for research | Sharing identifiable data without authorization | Safe Harbor: remove all 18 identifiers |
| 14 | EXIF metadata in medical images | GPS, device IDs, timestamps in metadata | Strip all EXIF before storage |

## 12. Quick Reference

### DO

- Encrypt ePHI at rest (AES-256) and in transit (TLS 1.2+)
- Assign unique user IDs to every person/system accessing ePHI
- Implement MFA for ePHI access
- Log every ePHI access: who, what, when, where, outcome
- Enforce automatic session timeout (15 minutes or less)
- Use POST for PHI data; never put PHI in URLs
- Apply RBAC with minimum necessary field filtering
- Sign BAAs with every third-party service handling PHI
- Encrypt all backups containing ePHI
- Strip EXIF metadata from medical images
- Implement "break glass" emergency access with audit trail
- Retain audit logs for 6+ years
- Map all ePHI locations via risk analysis
- De-identify using Safe Harbor (remove all 18 identifiers) for research
- Rate-limit PHI endpoints to detect bulk extraction
- Implement HMAC integrity checks on ePHI records
- Build breach detection and containment capabilities
- Send generic push notifications; require auth before showing PHI

### DON'T

- Don't log PHI content in application/error/debug logs
- Don't put PHI in URLs, query parameters, or fragments
- Don't use `SELECT *` on tables containing PHI
- Don't use shared accounts for ePHI access
- Don't send PHI through services lacking a signed BAA
- Don't store PHI in localStorage/sessionStorage/cookies unencrypted
- Don't include PHI in push notifications or email subject lines
- Don't send PHI in analytics events or tracking pixels
- Don't transmit PHI over plain HTTP
- Don't return PHI in error messages or stack traces
- Don't skip audit logging for any ePHI operation
- Don't store unencrypted backups of ePHI
- Don't cache PHI in unprotected layers (CDN, browser cache, shared Redis)
- Don't use SMS for PHI transmission (not end-to-end encrypted)
- Don't retain PHI longer than necessary; enforce data retention policies

## Resources

- [Technical Safeguards Code Examples](resources/technical-safeguards.md) — Access control, audit logging, integrity checks, authentication, transmission security, admin safeguard implementations
- [De-identification Code Examples](resources/deidentification.md) — Safe Harbor SQL views, PHI detection regex, date shifting, complete de-identification function, minimum necessary role-scoped queries
- [Breach Response Code Examples](resources/breach-response.md) — Detection patterns, containment procedures, breach log schema, notification tracking
