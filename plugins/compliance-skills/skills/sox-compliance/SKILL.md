---
name: sox-compliance
description: Use when building financial reporting systems, accounting software, ERP integrations, payment reconciliation, ledger systems, audit trails, or any code handling financial data at publicly traded companies — SOX Section 302/404 controls, ITGC, segregation of duties, change management, audit logging
---

# SOX (Sarbanes-Oxley) Compliance Coding Guidelines

## 1. Overview

The Sarbanes-Oxley Act of 2002 (SOX) was enacted after major corporate accounting scandals (Enron, WorldCom) to protect investors by improving the accuracy and reliability of corporate financial disclosures. It applies to all publicly traded companies in the United States (and foreign companies listed on US exchanges), as well as companies preparing for an IPO. Under SOX, CEOs and CFOs personally certify the accuracy of financial reports -- penalties for willful violations include fines up to $5 million and imprisonment up to 20 years. For developers, this means every system that feeds into financial reporting must have documented, tested internal controls with complete audit trails. SOX compliance is framework-based, built on COSO (Committee of Sponsoring Organizations -- 5 components, 17 principles for internal control) and COBIT (Control Objectives for Information and Related Technologies) for IT governance.

## 2. Key Sections for Developers

| Section | Requirement | Developer Impact |
|---------|-------------|------------------|
| Section 302 | CEO/CFO must certify accuracy of financial reports | Software producing financial reports must guarantee data integrity -- calculations must be verifiable, inputs validated, outputs reconciled |
| Section 404 | Internal Controls over Financial Reporting (ICFR) must be documented, tested, and audited annually | Every system in the financial reporting chain needs documented controls, automated tests proving those controls work, and audit evidence |
| Section 802 | Criminal penalties for altering, destroying, or concealing records | Financial records must be retained for 7 years minimum; audit logs must be immutable (append-only); penalties include up to 20 years imprisonment |
| Section 906 | Criminal penalties for false financial certification | Systems must produce accurate, complete, and timely financial data -- bugs that cause misstatement are compliance violations |

## 3. IT General Controls (ITGC) -- The Four Domains

### 3.1 Domain 1: Access to Programs and Data

Every financial system must enforce unique identification, least privilege, and periodic access review.

| Requirement | Description | Key Rules |
|-------------|-------------|-----------|
| Unique IDs | Every person has a unique user identity | No shared or generic accounts; service accounts scoped per application |
| RBAC | Role-based access with least privilege | Separate roles for Viewer, Data Entry, Approver, Reconciler, Admin, Auditor |
| SoD in roles | Conflicting permissions never assigned together | Data Entry cannot Approve; Admin cannot access financial data directly |
| Access reviews | Quarterly review of all financial system users | Flag inactive users (>90 days), remove terminated users same-day |
| Deprovisioning | Automated revocation on termination | Daily check for terminated employees with active roles; immediate revocation |
| Logging | All access granted/denied events recorded | Every authentication and authorization decision written to audit log |

### 3.2 Domain 2: Change Management

All changes to financial systems must be authorized, tested, and approved before reaching production.

| Requirement | Description | Key Rules |
|-------------|-------------|-----------|
| Change requests | Every change linked to approved ticket | CI/CD gate blocks deployment without valid ticket reference |
| Branch protection | No direct pushes to main branch | Require 2+ approvals, code owner review, signed commits, status checks |
| Code ownership | Financial modules require finance-team review | CODEOWNERS file maps financial paths to finance + security teams |
| PR traceability | Pull requests document SOX impact | Template includes ticket number, financial impact, SoD attestation, rollback plan |
| Deployment gates | Automated checks before production | Verify ticket approved, approver != author, tests passed, reviews complete |
| No manual deploys | All deployments through CI/CD pipeline | SSH deploy access restricted and logged; pipeline is sole mechanism |

### 3.3 Domain 3: Program Development

Security and compliance requirements must be part of the design phase, not afterthoughts.

| Requirement | Description |
|-------------|-------------|
| Security requirements | Documented before coding begins |
| Data flow diagrams | Show where financial data enters, moves, and is stored |
| Threat model | Identify risks to data integrity |
| Test plan | Cover unit, integration, and user acceptance testing |
| Test evidence | Retain test reports, screenshots, sign-off records |
| Go-live approval | Business owner approves (not the development team) |

### 3.4 Domain 4: Computer Operations

| Requirement | Description | Key Rules |
|-------------|-------------|-----------|
| Backup verification | Daily verification of financial database backups | Check existence, recency (<25 hours), checksum integrity, minimum size |
| Batch monitoring | Monitor critical financial batch processes | Alert on failure, late start, or exceeded duration for GL posting, sub-ledger sync, reconciliation, report generation |
| Restore testing | Quarterly full restore test with documented results | Verify RTO/RPO achievable; retain evidence |
| Incident response | Documented procedures for financial system outages | Escalation paths, communication plans, root cause analysis |

> **Code examples:** See [resources/itgc-patterns.md](resources/itgc-patterns.md) for RBAC middleware (Python/PHP), access review queries, deprovisioning automation, branch protection config, CODEOWNERS, PR templates, deployment gates, test documentation, backup verification, and batch monitoring patterns.

## 4. Segregation of Duties (SoD)

The four financial duties that must be separated so no single person controls an entire transaction:

| Duty | Description | System Role | Example |
|------|-------------|-------------|---------|
| Authorization | Approving transactions | Approver | Approve a journal entry for posting |
| Recording | Creating/modifying records | Data Entry | Create a journal entry |
| Custody | Responsibility for assets | Treasury | Execute a payment |
| Verification | Reconciling and reviewing | Reconciler | Verify bank statement matches ledger |

**Conflicting action pairs (must be enforced in code):**

| Action A | Cannot also perform | Reason |
|----------|---------------------|--------|
| `journal:create` | `journal:approve`, `journal:post` | Creator cannot approve or post own entry |
| `payment:request` | `payment:approve`, `payment:execute` | Requestor cannot approve or execute own payment |
| `reconciliation:perform` | `journal:create`, `journal:approve` | Reconciler is independent of recording and authorization |

**SoD rules for software development:**

| Rule | Enforcement |
|------|-------------|
| Author != Reviewer | Branch protection; CI verifies reviewer is not commit author |
| Author != Deploy Approver | CI/CD gate; production deploy approver cannot be PR author or last commit author |
| Dev != Prod Access | Developers have no direct production database access; read-only replicas for debugging |
| Automated deployment only | CI/CD pipeline is the sole deployment mechanism; no manual SSH deploys |

> **Code examples:** See [resources/itgc-patterns.md](resources/itgc-patterns.md) for SoD enforcement class (Python), CI/CD SoD pipeline config, and CODEOWNERS review separation.

## 5. Audit Trail Implementation

This is the single most critical coding pattern for SOX compliance. Every mutation to financial data must be captured in an immutable, tamper-evident log.

**Required fields for every audit record:**

| Field | Description | SOX Requirement |
|-------|-------------|-----------------|
| WHO | Authenticated user ID (unique, never shared) | Section 302 -- accountability |
| WHAT | Before and after values of every changed field | Section 404 -- evidence of controls |
| WHEN | Precise timestamp with timezone (UTC preferred) | Section 802 -- record integrity |
| WHERE | System name, module, table, record ID | Section 404 -- traceability |
| WHY | Linked change request or business justification | Section 404 -- authorization evidence |

**Key audit trail rules:**

| Rule | Detail |
|------|--------|
| Append-only | Audit log table permits INSERT and SELECT only; no UPDATE or DELETE |
| Hash chain | Each record includes SHA-256 of previous record for tamper detection |
| 7-year retention | Financial audit logs retained minimum 7 years (Section 802) |
| Partition by year | Partition tables for efficient archival without deletion |
| Periodic verification | Run hash chain verification regularly to detect tampering |
| Full state capture | Always record complete before and after state, not just changed fields |

> **Code examples:** See [resources/audit-trail.md](resources/audit-trail.md) for append-only SQL schema with hash chain, Python AuditLogger class with chain verification, PHP FinancialAuditLogger, retention partitioning, and chain verification patterns.

## 6. Application Controls

### 6.1 Input Controls

| Control | Description | Key Rules |
|---------|-------------|-----------|
| Completeness | All required fields present | Validate date, description, lines, created_by on every journal entry |
| Date validation | No future-dated entries without approval; no entries in closed periods | Check against period close calendar |
| Amount validation | Type-safe decimal handling; range checks | Max 2 decimal places; no negatives; configurable threshold limit |
| Balance check | Debits must equal credits | Reject unbalanced entries before saving |
| Account validation | Codes must exist in chart of accounts | Verify account exists and is active |
| Duplicate detection | Flag potential duplicate entries | Require override approval for duplicates |
| Currency validation | Only valid currencies accepted | Validate against allowed currency set |

### 6.2 Processing Controls

| Control | Description | Key Rules |
|---------|-------------|-----------|
| Approval workflow | Journal entries require approval before posting | Approver != creator (SoD); amount-based escalation thresholds |
| Transaction atomicity | All-or-nothing posting to general ledger | BEGIN/COMMIT/ROLLBACK; either all lines post or none |
| Reconciliation | Sub-ledger to GL automated reconciliation | Run daily; penny tolerance; alert finance team on discrepancy |
| Period controls | Prevent posting to closed periods | Only authorized users can reopen periods with audit trail |

**Approval escalation thresholds:**

| Amount Threshold | Required Approval Level |
|------------------|------------------------|
| < $10,000 | Supervisor |
| >= $10,000 | Controller |
| >= $100,000 | Controller |
| >= $1,000,000 | CFO |

### 6.3 Output Controls

| Control | Description | Key Rules |
|---------|-------------|-----------|
| Authorization | Only authorized recipients receive financial reports | Role-based report access; distribution lists reviewed quarterly |
| Reconciliation | Report totals verified against source data | Output-to-input reconciliation before distribution |
| Completeness | All expected accounts present in reports | Verify against expected account list; reject incomplete reports |
| Access logging | All report access recorded | Log who accessed which report, when |

> **Code examples:** See [resources/application-controls.md](resources/application-controls.md) for journal entry validator, approval workflow with SoD, transaction atomicity, sub-ledger reconciliation, and report authorization patterns.

## 7. Data Integrity Controls

| Control | Description | Key Rules |
|---------|-------------|-----------|
| Amount validation | Validate monetary amounts at every entry point | Decimal type (never float); max 2 decimal places; no negatives; required field |
| Account code validation | Validate against chart of accounts | Must exist; must be active; required field |
| Transfer checksums | Verify data integrity between systems | Record count + control total + SHA-256 hash; verify all three at destination |
| Encryption at rest | Encrypt sensitive financial data | Key management with rotation; least-privilege key access; log all decryption |

## 8. Change Management for Code

**The full SOX-compliant change pipeline -- ten required steps:**

| Step | Action | Key Rule |
|------|--------|----------|
| 1 | Change Request | Documented in ticketing system with business justification |
| 2 | Impact Assessment | Identify which financial reports/processes are affected |
| 3 | Approval | Business owner approves (NOT the developer) |
| 4 | Development | Isolated environment; no production data |
| 5 | Testing | Unit + integration + security + UAT |
| 6 | Code Review | By peer (NOT the author); minimum 2 reviewers |
| 7 | Deploy Approval | Separate from dev approval (SoD) |
| 8 | Automated Deploy | CI/CD pipeline (not manual) |
| 9 | Post-Deploy Verify | Smoke tests + reconciliation check |
| 10 | Audit Trail | Every step logged with who/what/when |

## 9. Backup and Recovery

| System | Frequency | Retention | Encryption | Offsite | Restore Test | RTO | RPO |
|--------|-----------|-----------|------------|---------|-------------|-----|-----|
| Financial database | Daily | 7 years | Yes | Yes | Quarterly | 4h | 1h |
| Audit log database | Daily | 7 years | Yes | Yes | Quarterly | 4h | 1h |
| Document storage | Daily | 7 years | Yes | Yes | Annually | 24h | 24h |

**Backup compliance verification checks:** existence, recency, encryption, offsite copy, checksum integrity, restore test currency.

> **Code examples:** See [resources/integrity-change-backup.md](resources/integrity-change-backup.md) for data validation, transfer checksums, encryption helpers, CI/CD pipeline config, deployment checklist, and backup compliance verification patterns.

## 10. Common Mistakes

| Mistake | SOX Risk | Fix |
|---------|----------|-----|
| Shared service accounts for financial systems | No individual accountability; audit trail is meaningless | Unique user IDs per person; service accounts scoped per application with logged usage |
| Developer self-approves and deploys own code | Segregation of duties violation | Branch protection requiring peer review; CI/CD deploys (never manual); deploy approver cannot be author |
| No audit trail on journal entry mutations | Cannot prove controls are operating; Section 802 violation | Append-only audit log on every create, update, approve, post, and void operation |
| Audit log captures action but not before/after values | Auditors cannot verify what changed; insufficient evidence | Always capture full before and after state of the affected record |
| Developers have direct production database access | Unauthorized changes possible without controls | Remove direct access; all changes through application layer with audit trail; read-only replicas for debugging |
| No change ticket linked to code deployments | Cannot prove changes were authorized | CI/CD gate that blocks deployment without valid, approved ticket reference |
| Hardcoded credentials in financial system code | Unauthorized access risk; credential rotation impossible | Environment variables or secrets manager; rotate credentials on schedule; never commit secrets |
| No backup restore testing | Backup may be corrupt; RTO/RPO unverifiable | Quarterly restore tests with documented results; automated backup integrity verification |
| Audit logs can be modified or deleted | Tamper risk destroys evidentiary value; Section 802 violation | Append-only table permissions (INSERT and SELECT only); hash chain for tamper detection |
| Journal entries post without approval workflow | No authorization control; Section 404 finding | Mandatory approval step with SoD (creator cannot approve); amount-based escalation thresholds |
| No environment segregation (dev/staging/prod) | Untested code can reach production; unauthorized changes | Separate environments; no production data in dev/test; pipeline enforces promotion path |
| Manual production deployments via SSH | No audit trail; no SoD; no testing gate | Automated CI/CD pipeline as the sole deployment mechanism; SSH access restricted and logged |
| No reconciliation between sub-ledger and GL | Data discrepancies go undetected | Automated daily reconciliation with alerting on discrepancies; monthly management review |
| Financial reports accessible to all users | Unauthorized disclosure of material non-public information | Role-based report access; distribution lists reviewed quarterly; access logged |
| Logs do not include timestamps with timezone | Cannot establish event sequence for investigations | UTC timestamps with millisecond precision on all audit records |

## 11. Quick Reference

### DO

- Use unique user IDs for every person accessing financial systems (SOX 302)
- Implement RBAC with least privilege on all financial endpoints (ITGC Domain 1)
- Enforce segregation of duties: creator cannot approve, author cannot deploy (SOX 404)
- Log every financial data mutation with WHO, WHAT (before/after), WHEN, WHERE, WHY (SOX 802)
- Use append-only audit tables with hash chains for tamper detection (SOX 802)
- Retain financial records and audit logs for a minimum of 7 years (SOX 802)
- Validate all financial inputs: type, range, format, completeness, balance (SOX 404)
- Require approval workflows for journal entries with amount-based escalation (SOX 404)
- Use transaction atomicity (BEGIN/COMMIT/ROLLBACK) for all financial postings (SOX 404)
- Reconcile sub-ledgers to general ledger daily with automated alerting (SOX 404)
- Link every code deployment to an approved change request (ITGC Domain 2)
- Require peer code review (minimum 2 reviewers, not the author) (ITGC Domain 2)
- Deploy only through automated CI/CD pipelines, never manually (ITGC Domain 2)
- Encrypt financial data at rest and in transit (SOX 404)
- Test backups quarterly with documented restore results (ITGC Domain 4)
- Run periodic access reviews and remove terminated user access promptly (ITGC Domain 1)
- Document all controls and retain test evidence for auditors (SOX 404)
- Verify report output totals against source data before distribution (SOX 404)
- Use separate dev, staging, and production environments (ITGC Domain 2)
- Implement checksums for data transfers between financial systems (SOX 404)

### DO NOT

- Use shared or generic accounts on any financial system (violates SOX 302)
- Allow the same person to create AND approve journal entries (SoD violation)
- Allow code authors to approve or deploy their own changes (SoD violation)
- Store financial data without encryption at rest (SOX 404 finding)
- Permit UPDATE or DELETE operations on audit log tables (SOX 802 violation)
- Deploy code without an approved change ticket (ITGC Domain 2 finding)
- Grant developers direct access to production databases (ITGC Domain 1 finding)
- Skip before/after value capture in audit logs (insufficient audit evidence)
- Use manual deployment processes for financial systems (no audit trail, no SoD)
- Allow financial reports to be accessed without authorization checks (SOX 302)
- Delete or modify audit records under any circumstances (SOX 802, up to 20 years prison)
- Use production financial data in development or test environments (data leakage risk)
- Deploy without passing all automated tests (ITGC Domain 2 finding)
- Skip backup integrity verification (ITGC Domain 4 finding)
- Hardcode credentials or secrets in source code (ITGC Domain 1 finding)
- Allow entries in closed accounting periods without documented override (SOX 404)
- Ignore reconciliation discrepancies between systems (SOX 404 finding)
- Build financial logic without documented test cases (ITGC Domain 3 finding)

## Resources

Detailed code examples and implementation patterns for each domain:

- [ITGC Patterns](resources/itgc-patterns.md) -- RBAC middleware, access reviews, deprovisioning, branch protection, deployment gates, SoD enforcement
- [Audit Trail Implementation](resources/audit-trail.md) -- Append-only schema, hash chain, Python/PHP loggers, retention partitioning, chain verification
- [Application Controls](resources/application-controls.md) -- Input validation, approval workflows, transaction atomicity, reconciliation, report authorization
- [Integrity, Change Management, and Backup](resources/integrity-change-backup.md) -- Data validation, transfer checksums, encryption, CI/CD pipeline, deployment checklist, backup verification
