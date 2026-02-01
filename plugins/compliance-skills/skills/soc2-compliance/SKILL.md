---
name: soc2-compliance
description: Use when building SaaS platforms, cloud services, customer-facing APIs, multi-tenant systems, or any service handling customer data that requires SOC 2 certification — Trust Services Criteria, TSC, CC1 through CC9, Type I, Type II, AICPA, security, availability, processing integrity, confidentiality, privacy, evidence collection, vendor management, penetration testing, risk assessment, access review, change management, incident response
---

# SOC 2 Compliance Coding Guidelines

## 1. Overview

SOC 2 (Service Organization Control 2) is an auditing framework developed by the AICPA for service organizations that store, process, or transmit customer data. It evaluates controls based on five Trust Services Criteria: Security, Availability, Processing Integrity, Confidentiality, and Privacy. SOC 2 Type I assesses control design at a single point in time; Type II evaluates operating effectiveness over a 3-to-12-month observation period and is the standard customers demand. Audits typically cost $20K-$100K+ depending on scope and complexity. The core principle for developers: "Say something, do something, prove you always do what you say." Every control you implement must be documented, consistently enforced, and produce evidence that auditors can verify.

## 2. The 5 Trust Services Criteria

| Criterion | Required? | What Developers Must Do |
|---|---|---|
| **Security** (CC1-CC9) | REQUIRED for all SOC 2 | Implement access controls, encrypt data, monitor systems, follow change management, build incident response capabilities |
| **Availability** | Optional | Build uptime monitoring, disaster recovery, backup automation, health checks, capacity planning |
| **Processing Integrity** | Optional | Validate all inputs, handle errors completely, ensure transaction atomicity, reconcile data between systems |
| **Confidentiality** | Optional | Classify data, encrypt at rest and in transit, enforce retention policies, implement secure deletion |
| **Privacy** | Optional | Implement consent management, data minimization, disclosure controls, subject access/correction endpoints |

Security is always in scope. The other four are selected based on what your service does. A SaaS platform handling customer data typically includes Security + Availability + Confidentiality at minimum.

## 3. Common Criteria (CC1-CC9) — Security Deep Dive

| CC | Name | Developer Responsibility |
|---|---|---|
| CC1 | Control Environment | Understand and follow security policies; complete required security training; acknowledge acceptable use policies |
| CC2 | Communication & Information | Document system architecture, data flows, and security procedures; maintain runbooks in version control |
| CC3 | Risk Assessment | Participate in risk assessments; maintain asset and data inventory; flag new risks during design reviews |
| CC4 | Monitoring Activities | Implement monitoring dashboards; review control effectiveness quarterly; report control failures |
| CC5 | Control Activities | Implement technical controls (auth, encryption, validation); automate enforcement where possible |
| CC6 | Logical & Physical Access | RBAC, MFA, encryption, access reviews, network segmentation, least privilege, session management |
| CC7 | System Operations | Monitoring, alerting, incident detection and response, disaster recovery, backup verification |
| CC8 | Change Management | Branch protection, code review, CI/CD gates, deployment tracking, rollback procedures, segregation of duties |
| CC9 | Risk Mitigation | Vendor assessment, third-party risk management, business continuity, insurance, contractual protections |

## 4. Access Controls (CC6)

### Key Principles

- **RBAC over user checks** — Define roles with explicit permissions. Never check for specific users; always check roles or permissions.
- **Least privilege** — Start with zero access, grant only what is needed. Scoped API keys, not global keys.
- **Audit everything** — Log every authorization decision (granted and denied) with user ID, resource, permission, IP, and timestamp.
- **Session limits** — 15-minute idle timeout, 12-hour absolute timeout, maximum 3 concurrent sessions per user, secure cookie flags (HttpOnly, Secure, SameSite=Strict).
- **MFA for sensitive actions** — Require MFA for admin operations, data export, role changes, and API key management.
- **API key hygiene** — Hash keys at rest (never store plaintext), enforce expiration (90 days max), scope to specific permissions, show raw key only once at creation.
- **Quarterly access reviews** — Export active users with roles and last login; flag accounts inactive 90+ days for deprovisioning; audit all permission changes.
- **Deprovisioning checklist** — On termination: deactivate account, terminate all sessions, revoke all API keys, revoke OAuth tokens, remove from all groups. Log every step.

> Implementation code: RBAC middleware (Python, Node.js), session management, API key lifecycle, access review SQL queries, deprovisioning automation — see [resources/access-encryption.md](resources/access-encryption.md)

## 5. Encryption Standards

### Requirements

| What | Algorithm / Standard | Minimum |
|---|---|---|
| Data at rest | AES-256 (via Fernet/PBKDF2 or equivalent) | 256-bit key, 480K+ PBKDF2 iterations |
| Data in transit | TLS 1.2+ | Strong cipher suites only (AES-GCM, ChaCha20) |
| HSTS | Strict-Transport-Security | max-age=31536000, includeSubDomains, preload |
| Internal service-to-service | mTLS with certificate verification | TLS 1.2+, verify CA |
| Key rotation | Decrypt with old key, re-encrypt with new key | Defined rotation schedule |
| Secrets management | Vault service or environment variables | Never hardcode; never commit .env files |

### Anti-patterns

- **Never** hardcode secrets: `DATABASE_URL = "postgresql://admin:s3cr3t@..."` is a CC6 violation.
- **Never** commit `.env`, `*.pem`, `*.key`, or `credentials.*` files. Enforce via `.gitignore`.
- Always load secrets from vault at runtime, fall back to environment variables.

> Implementation code: AES-256 encrypt/decrypt with key rotation, TLS config, vault-based secrets loading (Python, Node.js) — see [resources/access-encryption.md](resources/access-encryption.md)

## 6. Logging and Monitoring (CC7)

### What to Log

| Event Category | Specific Events |
|---|---|
| Authentication | Login success, login failure, logout, MFA challenge, MFA failure, password reset, account lockout |
| Authorization | Permission granted, permission denied, role change, privilege escalation |
| Data Access | Customer data read, data export, data download, bulk query, API data access |
| Data Modification | Record created, updated, deleted, bulk operations, schema changes |
| Configuration | Setting changed, feature toggled, threshold modified, policy updated |
| Administrative | User created, user deactivated, role assigned, API key issued/revoked |
| System | Service start/stop, deployment, health check failure, dependency failure |
| Security | Suspicious activity, rate limit hit, blocked request, invalid token |

### Log Requirements

- **Format**: Structured JSON with timestamp, service, event, actor (user_id, IP, user_agent, session_id), resource (type, id), action, outcome, request_id.
- **Retention**: Audit logs 365 days minimum, application logs 90 days, security logs 365 days with real-time alerting.
- **Immutability**: Append-only storage. No edits or deletes. Encrypted at rest (AES-256). Restricted access (admin + auditor only).

> Implementation code: Structured audit logger (Python, Node.js), log retention policy config — see [resources/logging-change-incident.md](resources/logging-change-incident.md)

## 7. Change Management (CC8)

### Branch Protection Rules

- At least 1 approving review required; dismiss stale reviews; require code owner reviews.
- Required status checks: unit tests, integration tests, security lint. Branch must be up to date.
- Enforce for admins — no bypasses. No direct push. No force pushes. No deletions. Linear history required.

### Deployment Requirements

- Every deployment links to a tracked change request (ticket/PR/CR).
- Author and approver must be different people (segregation of duties).
- All tests must pass before deployment. Rollback plan required.
- Deployment record includes: deployment_id, environment, timestamp, deployer (service account), change_request, PR, commit SHA, approver, author, tests_passed, rollback_plan, changes_summary.

### Emergency Changes

- Allowed only for: active security incidents, customer-affecting outages, production data integrity issues.
- Requires verbal approval from on-call lead (documented within 24 hours), post-deployment review within 48 hours, retroactive change request ticket, and root cause analysis.

> Implementation code: Branch protection YAML, deployment validation, emergency change policy — see [resources/logging-change-incident.md](resources/logging-change-incident.md)

## 8. Incident Response (CC7)

### Severity Classification

| Level | Name | Response Time | Description |
|---|---|---|---|
| P1 | Critical | 15 minutes | Active breach, data loss, or complete service outage |
| P2 | High | 1 hour | Security vulnerability exploited, partial outage, data at risk |
| P3 | Medium | 4 hours | Vulnerability found but not exploited, degraded performance |
| P4 | Low | 1 business day | Minor issue, policy violation, improvement needed |

### Key Components

- **Health checks**: Component-level status endpoints (database, cache, storage, external APIs) with latency measurement.
- **Anomaly detection**: Thresholds for failed logins (10/user/hour, 50/IP/hour), API errors (100/min), data exports (500MB), admin actions (50/hour), after-hours admin logins (1).
- **Containment automation**: On suspected compromise, immediately disable account, terminate sessions, revoke all credentials, block suspicious IPs, preserve evidence.
- **Post-incident review**: Timeline (detected, acknowledged, contained, resolved, post-mortem), impact assessment (customers affected, data exposed, downtime), root cause, remediation (immediate + long-term), lessons learned, action items with owners.

> Implementation code: Health check endpoints, anomaly detection, severity definitions, containment automation, post-incident template — see [resources/logging-change-incident.md](resources/logging-change-incident.md)

## 9. Availability Controls

### Key Patterns

- **Circuit breaker**: Track failures per dependency. After threshold (e.g., 5 failures), open circuit and reject requests immediately. After recovery timeout, allow test requests. Close circuit after success threshold met.
- **Retry with exponential backoff**: Max 3 retries, base delay 1s, max delay 30s, jitter to prevent thundering herd. Only retry on transient errors (502, 503, 504, ECONNRESET, ETIMEDOUT).
- **Backup verification**: Monthly restoration tests to isolated environment. Verify checksum, restore, validate data integrity (record counts + sample records). Log results as SOC 2 evidence.

> Implementation code: Circuit breaker (Python), retry with backoff (Node.js), backup verification — see [resources/availability-integrity.md](resources/availability-integrity.md)

## 10. Processing Integrity Controls

### Key Patterns

- **Input validation**: Validate at every API boundary. Use schema validation (Pydantic, Joi, etc.). Reject invalid data early. Log validation failures with endpoint and error details.
- **Idempotency**: State-changing operations must accept an idempotency key. Check for existing result before processing. Store results with 24-hour TTL. Log both new processing and replays.
- **Data reconciliation**: Periodic cross-system checks (e.g., usage records vs invoices). Flag discrepancies exceeding threshold (e.g., $0.01). Alert on mismatches. Log reconciliation results as evidence.

> Implementation code: Input validation (Python Pydantic, Node.js), idempotency pattern, billing reconciliation — see [resources/availability-integrity.md](resources/availability-integrity.md)

## 11. Data Handling

### 11.1 Data Classification

| Level | Examples | Required Controls |
|---|---|---|
| **Public** | Marketing pages, public docs, blog posts | No special controls |
| **Internal** | Employee directory, internal metrics, sprint boards | Access control, authentication required |
| **Confidential** | Customer data, financial records, contracts, analytics | Encryption at rest + in transit, access control, audit logging, retention policy |
| **Restricted** | Credentials, PII, health records, payment card data, encryption keys | All of Confidential + MFA for access, strict need-to-know, automated retention enforcement, secure deletion |

### 11.2 Retention and Deletion

| Data Type | Retention | Deletion Method |
|---|---|---|
| Customer data | While account active + 30 days grace | Hard delete with verification |
| Audit logs | 365 days minimum | Automated after retention |
| Session data | 24 hours after expiry | Automated cleanup |
| Backups | 90 days | Automated with verification |

- On customer deletion: remove from all tables (records, files, metadata, API keys, sessions), verify zero remaining rows, log with full audit trail.
- **Test data**: Never use production customer data in non-production environments. Generate synthetic data or properly anonymize (mask names, emails, phones, addresses; preserve structural non-PII fields).

## 12. Vendor Management (CC9)

### Vendor Registry Requirements

Every third-party vendor accessing customer data must have a registry entry tracking:
- Vendor name, service provided, data access level, data classification
- SOC 2 report status (type, date, findings, next review)
- Contract clauses: data protection, audit rights, breach notification SLA, deletion on termination, subprocessor notification
- Risk rating (low/medium/high/critical), review dates, owner

### Security Requirements Checklist

- SOC 2 Type II report available and reviewed
- Data encryption at rest (AES-256 or equivalent)
- Data encryption in transit (TLS 1.2+)
- Access controls with least privilege
- Incident response plan with notification SLA
- Business continuity and disaster recovery plan
- Background checks for employees accessing data
- Data deletion capability upon contract termination
- Subprocessor disclosure and notification
- Right to audit clause in contract
- Data residency and jurisdiction compliance
- Vulnerability management program

Annual vendor security assessments are required. Log assessment initiation and results.

## 13. Type I vs Type II — Developer Impact

| Aspect | Type I | Type II |
|---|---|---|
| **Scope** | Control design at a single point in time | Operating effectiveness over 3-12 months |
| **Evidence needed** | Policies, configurations, screenshots, architecture diagrams | Everything in Type I + logs, incident records, review records, change history over time |
| **Developer burden** | Implement controls, write policies, configure systems | Continuously operate controls, generate evidence, respond to findings |
| **Audit duration** | 2-4 weeks | 6-12 weeks (plus observation period) |
| **What breaks it** | Missing controls, incomplete policies | Controls that exist but are not consistently followed |
| **Typical timeline** | 2-3 months to prepare | 3-6 months prep + 3-12 months observation |
| **Key developer tasks** | Set up logging, RBAC, encryption, branch protection, monitoring | Prove you did quarterly access reviews, responded to incidents within SLA, never self-merged, rotated keys on schedule |

Type I answers: "Are your controls designed properly?"
Type II answers: "Do your controls actually work, consistently, over time?"

## 14. Evidence Generation Patterns

Auditors typically request 200-300 evidence items. Automate collection wherever possible.

### Evidence Categories

| Category | Key Sources | Automatable? |
|---|---|---|
| Access controls | User/role exports, access review records, deprovisioned user list, MFA report, API key inventory | Yes |
| Change management | Branch protection config, PR history with reviewers, deployment log, emergency changes, rollbacks | Yes |
| Incident response | Incident tickets with timelines, post-incident reviews, annual IR test, alert configs, on-call schedule | Partial |
| Monitoring | Uptime reports, alert config exports, dashboard screenshots, log retention config, anomaly rules | Yes |
| Encryption | TLS config, certificate inventory, encryption-at-rest config, key rotation records | Partial |
| Vendor management | Vendor inventory, SOC 2 reports on file, annual review records, contract reviews | No |

> Implementation code: Evidence category map, automated quarterly export function — see [resources/evidence-collection.md](resources/evidence-collection.md)

## 15. Common Mistakes

| Mistake | TSC / CC Violation | Fix |
|---|---|---|
| Hardcoded secrets in source code | CC6 (Access Controls) | Use vault service or environment variables; scan repos for leaked secrets |
| Self-merged pull requests | CC8 (Change Management) | Enforce branch protection: author cannot be approver |
| No deployment tracking | CC8 (Change Management) | Log every deployment with commit SHA, approver, ticket link |
| No quarterly access reviews | CC6 (Access Controls) | Schedule automated access reports; review and sign off quarterly |
| Unencrypted customer data at rest | CC6 (Access Controls), Confidentiality | AES-256 encryption for all customer data in databases and storage |
| No log retention policy | CC7 (System Operations) | Define retention (1 year minimum for audit logs), configure immutable storage |
| Shared service accounts | CC6 (Access Controls) | Unique service accounts per application/function with minimum required scope |
| No incident response plan | CC7 (System Operations) | Document and test IR plan annually; define severity levels and response SLAs |
| Production customer data in test environments | Confidentiality, Privacy | Use synthetic data or properly anonymized datasets |
| No vendor inventory | CC9 (Risk Mitigation) | Maintain registry of all third parties; review SOC 2 reports annually |
| Manual deployments without audit trail | CC8 (Change Management) | Automate deployments; log who deployed what, when, and why |
| Missing input validation | Processing Integrity | Validate at every API boundary; reject malformed data early |
| No backup testing | Availability | Test backup restoration monthly; log results as evidence |
| No session timeout | CC6 (Access Controls) | 15-minute idle timeout, 12-hour absolute timeout |
| Missing error handling | Processing Integrity | Catch all errors; log with correlation IDs; never swallow exceptions silently |
| Logging PII in plain text | Confidentiality, Privacy | Mask or hash PII in logs; log user IDs instead of names/emails |
| No MFA for admin accounts | CC6 (Access Controls) | Require MFA for all privileged access; enforce via application logic |
| Permissive CORS configuration | CC6 (Access Controls) | Whitelist specific origins; never use `Access-Control-Allow-Origin: *` for authenticated endpoints |

## 16. Quick Reference

### DO

- Encrypt all customer data at rest (AES-256) and in transit (TLS 1.2+)
- Implement RBAC with least privilege — check permissions, not usernames
- Log all authentication, authorization, data access, and configuration changes
- Use structured JSON logs with timestamps, user IDs, IPs, and request IDs
- Require peer review for all code changes — no self-merges
- Link every deployment to a tracked change request
- Rotate secrets and API keys on a defined schedule (90 days maximum)
- Set session timeouts (15-minute idle, 12-hour absolute)
- Validate all inputs at API boundaries
- Use idempotency keys for state-changing operations
- Test backup restoration monthly and log results
- Run quarterly access reviews and deactivate unused accounts
- Maintain a vendor inventory with annual security reviews
- Classify data and apply controls matching the classification level
- Generate automated evidence exports for auditors
- Document and test incident response procedures annually
- Use unique service accounts with minimum necessary scope
- Mask or exclude PII from log entries

### DO NOT

- Hardcode secrets, API keys, or credentials in source code
- Use production customer data in test or staging environments
- Allow self-approval of code changes or deployments
- Deploy without automated tests passing
- Share service accounts or credentials between people or systems
- Store logs in mutable storage where entries can be edited or deleted
- Skip input validation because "the frontend handles it"
- Swallow exceptions without logging them
- Use `Access-Control-Allow-Origin: *` on authenticated endpoints
- Grant admin access by default — start with minimum permissions
- Ignore failed login attempts — alert on brute force patterns
- Delete audit logs before the retention period expires
- Skip vendor security reviews because "they are a big company"
- Deploy emergency changes without retroactive documentation
- Assume encryption is handled by the infrastructure — verify it
- Log raw credentials, tokens, or full credit card numbers

## Resources

Detailed implementation code for each control area:

- **[resources/access-encryption.md](resources/access-encryption.md)** — RBAC middleware, session management, API key lifecycle, access review queries, user deprovisioning, AES-256 encryption, TLS configuration, secrets management
- **[resources/logging-change-incident.md](resources/logging-change-incident.md)** — Structured audit logger, log retention config, branch protection YAML, deployment tracking, emergency change policy, health checks, anomaly detection, severity classification, containment automation, post-incident review template
- **[resources/availability-integrity.md](resources/availability-integrity.md)** — Circuit breaker pattern, retry with exponential backoff, backup verification, input validation, idempotency, data reconciliation
- **[resources/evidence-collection.md](resources/evidence-collection.md)** — Evidence category map, automated quarterly evidence export
