# Evidence Collection — Implementation Code

## Evidence Category Map

```python
EVIDENCE_MAP = {
    "access_controls": {
        "sources": [
            "User list with roles and last login dates (quarterly export)",
            "Access review records (quarterly)",
            "Deprovisioned user list with dates",
            "MFA enrollment report",
            "API key inventory with expiration dates",
            "Service account inventory",
        ],
        "automated": True,
        "query": "SELECT id, email, role, mfa_enabled, last_login_at, "
                 "created_at FROM users WHERE is_active = TRUE",
    },
    "change_management": {
        "sources": [
            "Branch protection configuration export",
            "Pull request history with reviewers",
            "Deployment log with approvals",
            "Emergency change records",
            "Rollback history",
        ],
        "automated": True,
        "source_system": "git_and_deployment_pipeline",
    },
    "incident_response": {
        "sources": [
            "Incident tickets with timelines",
            "Post-incident reviews",
            "Annual IR test results",
            "Alert configurations",
            "On-call rotation schedule",
        ],
        "automated": "partial",
    },
    "monitoring": {
        "sources": [
            "Uptime reports",
            "Alert configuration exports",
            "Dashboard screenshots",
            "Log retention configuration",
            "Anomaly detection rules",
        ],
        "automated": True,
    },
    "encryption": {
        "sources": [
            "TLS configuration and certificate inventory",
            "Encryption-at-rest configuration",
            "Key rotation records",
            "Key management policy",
        ],
        "automated": "partial",
    },
    "vendor_management": {
        "sources": [
            "Vendor inventory with risk ratings",
            "Vendor SOC 2 reports on file",
            "Annual vendor review records",
            "Contract review records",
        ],
        "automated": False,
    },
}
```

## Automated Quarterly Evidence Export

```python
def generate_quarterly_evidence():
    """
    Automated quarterly evidence package for SOC 2 audit.
    Run via scheduled job; store output in evidence repository.
    """
    quarter = get_current_quarter()
    evidence = {}

    # Access control evidence
    evidence["active_users"] = db.query(
        "SELECT id, email, role, mfa_enabled, last_login_at, created_at "
        "FROM users WHERE is_active = TRUE ORDER BY role, email")

    evidence["access_reviews"] = db.query(
        "SELECT * FROM access_review_records "
        "WHERE review_date >= %s ORDER BY review_date DESC",
        [quarter.start])

    evidence["deprovisioned_users"] = db.query(
        "SELECT id, email, deactivated_at, deactivated_by, reason "
        "FROM users WHERE is_active = FALSE AND deactivated_at >= %s",
        [quarter.start])

    # Change management evidence
    evidence["deployments"] = db.query(
        "SELECT * FROM deployment_log "
        "WHERE deployed_at >= %s ORDER BY deployed_at DESC",
        [quarter.start])

    evidence["emergency_changes"] = db.query(
        "SELECT * FROM change_requests "
        "WHERE type = 'emergency' AND created_at >= %s",
        [quarter.start])

    # Incident evidence
    evidence["incidents"] = db.query(
        "SELECT * FROM incidents WHERE detected_at >= %s",
        [quarter.start])

    # Store with timestamp for audit trail
    store_evidence(quarter=quarter.label, data=evidence,
                   generated_at=utcnow().isoformat())

    audit_log("evidence_generated", quarter=quarter.label,
              categories=list(evidence.keys()))
```
