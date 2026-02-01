# Logging, Change Management & Incident Response — Implementation Code

## Structured Audit Logger (Python)

```python
import json
import logging
from datetime import datetime, timezone

class SOC2AuditLogger:
    """Structured audit logger meeting SOC 2 evidence requirements."""

    def __init__(self, service_name: str):
        self.service_name = service_name
        self.logger = logging.getLogger("audit")

    def log(self, event: str, **kwargs):
        entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "service": self.service_name,
            "event": event,
            "level": kwargs.pop("level", "info"),
            "actor": {
                "user_id": kwargs.pop("user_id", None),
                "ip_address": kwargs.pop("ip", None),
                "user_agent": kwargs.pop("user_agent", None),
                "session_id": kwargs.pop("session_id", None),
            },
            "resource": {
                "type": kwargs.pop("resource_type", None),
                "id": kwargs.pop("resource_id", None),
            },
            "action": kwargs.pop("action", event),
            "outcome": kwargs.pop("outcome", "success"),
            "request_id": kwargs.pop("request_id", None),
            "details": kwargs,  # Remaining fields
        }
        # Remove None values for cleaner logs
        entry = {k: v for k, v in entry.items() if v is not None}
        self.logger.info(json.dumps(entry, default=str))

audit = SOC2AuditLogger("my-service")

# Usage examples:
audit.log("user_login",
    user_id="usr_123", ip="203.0.113.42", outcome="success",
    user_agent="Mozilla/5.0", session_id="sess_abc")

audit.log("data_access",
    user_id="usr_123", ip="203.0.113.42",
    resource_type="customer_record", resource_id="cust_456",
    action="read", fields_accessed=["name", "email", "plan"])

audit.log("permission_denied",
    user_id="usr_789", ip="198.51.100.1",
    resource_type="admin_panel", action="access",
    outcome="denied", reason="insufficient_role",
    level="warning")
```

## Structured Audit Logger (Node.js)

```javascript
class AuditLogger {
  constructor(serviceName) {
    this.serviceName = serviceName;
  }

  log(event, data = {}) {
    const entry = {
      timestamp: new Date().toISOString(),
      service: this.serviceName,
      event,
      level: data.level || "info",
      actor: {
        userId: data.userId,
        ip: data.ip,
        userAgent: data.userAgent,
        sessionId: data.sessionId,
      },
      resource: {
        type: data.resourceType,
        id: data.resourceId,
      },
      action: data.action || event,
      outcome: data.outcome || "success",
      requestId: data.requestId,
      details: data.details || {},
    };
    // Ship to log aggregation — stdout for container-based collection
    process.stdout.write(JSON.stringify(entry) + "\n");
  }
}

const audit = new AuditLogger("my-service");

audit.log("config_changed", {
  userId: "usr_admin",
  ip: "10.0.0.1",
  resourceType: "system_setting",
  resourceId: "session_timeout",
  details: { oldValue: 30, newValue: 15 },
});
```

## Log Retention and Immutability

```python
LOG_RETENTION_POLICY = {
    "audit_logs": {
        "retention_days": 365,             # CC7: 1-year minimum retention
        "storage": "append_only",          # CC7: Immutable — no edits or deletes
        "encryption": "AES-256",           # CC6: Encrypted at rest
        "access_control": ["admin", "auditor"],  # CC6: Restricted access
    },
    "application_logs": {
        "retention_days": 90,
        "storage": "append_only",
    },
    "security_logs": {
        "retention_days": 365,
        "storage": "append_only",
        "alerting": True,                  # CC7: Real-time alerting on security events
    },
}
```

## Branch Protection Configuration

```yaml
# Git branch protection rules — enforce in your repository settings
# These configurations produce auditable evidence of CC8 compliance

protected_branches:
  main:
    required_pull_request_reviews:
      required_approving_review_count: 1    # At minimum one peer review
      dismiss_stale_reviews: true           # Re-review after new commits
      require_code_owner_reviews: true
    required_status_checks:
      strict: true                          # Branch must be up to date
      contexts:
        - "unit-tests"
        - "integration-tests"
        - "security-lint"
    enforce_admins: true                    # No bypasses, even for admins
    restrictions:
      users: []                             # No direct push
    allow_force_pushes: false
    allow_deletions: false
    require_linear_history: true            # No merge commits — clean audit trail
```

## Deployment Tracking and SoD Validation

```python
# Every deployment must link to a tracked change (ticket, PR, or CR)
DEPLOYMENT_RECORD = {
    "deployment_id": "deploy_20250115_001",
    "environment": "production",
    "deployed_at": "2025-01-15T14:30:00Z",
    "deployed_by": "deployer_service_account",   # Not a personal account
    "change_request": "CR-1234",                 # Linked tracking ticket
    "pull_request": "PR-567",
    "commit_sha": "abc123def456",
    "approved_by": "usr_reviewer",               # Different from author (CC8)
    "author": "usr_developer",                   # Different from approver (CC8)
    "tests_passed": True,
    "rollback_plan": "Revert commit abc123 and redeploy previous SHA",
    "changes_summary": "Updated session timeout from 30 to 15 minutes",
}

def validate_deployment(record):
    """Gate deployment on CC8 requirements."""
    errors = []
    if record["author"] == record["approved_by"]:
        errors.append("CC8: Author cannot approve their own change")
    if not record["tests_passed"]:
        errors.append("CC8: All tests must pass before deployment")
    if not record["change_request"]:
        errors.append("CC8: Deployment must link to a tracked change request")
    if not record["rollback_plan"]:
        errors.append("CC8: Rollback plan required")
    if errors:
        raise DeploymentBlockedError(errors)
```

## Emergency Change Procedure

```python
EMERGENCY_CHANGE_POLICY = {
    "requires": [
        "Verbal approval from on-call lead (documented within 24 hours)",
        "Post-deployment review within 48 hours",
        "Retroactive change request ticket created",
        "Root cause analysis completed",
    ],
    "allowed_when": [
        "Active security incident",
        "Service outage affecting customers",
        "Data integrity issue in production",
    ],
    "documentation": "Emergency changes must be reviewed and documented "
                     "retroactively within 48 hours with full audit trail.",
}
```

## Health Check Endpoints

```python
from datetime import datetime, timezone

def health_check():
    """
    Health endpoint for monitoring systems.
    Returns component-level status for incident detection.
    """
    checks = {
        "database": check_database_connection(),
        "cache": check_cache_connection(),
        "storage": check_storage_access(),
        "external_api": check_external_dependencies(),
    }
    all_healthy = all(c["status"] == "healthy" for c in checks.values())
    return {
        "status": "healthy" if all_healthy else "degraded",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "version": get_deployed_version(),
        "checks": checks,
    }

def check_database_connection():
    try:
        start = datetime.now(timezone.utc)
        db.execute("SELECT 1")
        latency_ms = (datetime.now(timezone.utc) - start).total_seconds() * 1000
        return {"status": "healthy", "latency_ms": round(latency_ms, 2)}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

## Anomaly Detection Patterns

```python
ALERT_THRESHOLDS = {
    "failed_logins_per_user_per_hour": 10,       # Brute force detection
    "failed_logins_per_ip_per_hour": 50,          # Credential stuffing
    "api_errors_per_minute": 100,                 # System instability
    "data_export_size_mb": 500,                   # Data exfiltration
    "admin_actions_per_hour": 50,                 # Compromised admin account
    "new_api_keys_per_day": 10,                   # Unauthorized automation
    "after_hours_admin_logins": 1,                # Suspicious timing
}

def check_anomaly(metric_name, current_value, context):
    threshold = ALERT_THRESHOLDS.get(metric_name)
    if threshold and current_value > threshold:
        alert = {
            "type": "anomaly_detected",
            "metric": metric_name,
            "value": current_value,
            "threshold": threshold,
            "context": context,
            "timestamp": utcnow().isoformat(),
            "severity": classify_severity(metric_name, current_value, threshold),
        }
        send_alert(alert)
        audit_log("security_anomaly", **alert)
```

## Severity Classification

```python
SEVERITY_LEVELS = {
    "P1": {
        "name": "Critical",
        "description": "Active breach, data loss, or complete service outage",
        "response_time": "15 minutes",
        "examples": ["Confirmed data breach", "All customers affected",
                      "Credentials exposed publicly"],
        "actions": ["Page on-call team", "Notify leadership", "Begin containment"],
    },
    "P2": {
        "name": "High",
        "description": "Security vulnerability exploited, partial outage, data at risk",
        "response_time": "1 hour",
        "examples": ["Active exploitation attempt", "Single-tenant data exposure",
                      "Authentication bypass discovered"],
        "actions": ["Alert security team", "Begin investigation"],
    },
    "P3": {
        "name": "Medium",
        "description": "Vulnerability found but not exploited, degraded performance",
        "response_time": "4 hours",
        "examples": ["Unpatched vulnerability", "Elevated error rates",
                      "Suspicious but unconfirmed activity"],
    },
    "P4": {
        "name": "Low",
        "description": "Minor issue, policy violation, improvement needed",
        "response_time": "1 business day",
        "examples": ["Failed access review followup", "Minor config drift",
                      "Documentation gap"],
    },
}
```

## Containment Automation

```python
def contain_compromised_account(user_id, incident_id, actor_id):
    """Automated containment for suspected account compromise."""
    actions_taken = []

    # 1. Disable the account
    disable_account(user_id)
    actions_taken.append("account_disabled")

    # 2. Terminate all active sessions
    count = terminate_all_sessions(user_id)
    actions_taken.append(f"sessions_terminated:{count}")

    # 3. Revoke all API keys and tokens
    revoke_all_api_keys(user_id)
    revoke_all_oauth_tokens(user_id)
    actions_taken.append("credentials_revoked")

    # 4. Block source IPs associated with suspicious activity
    suspicious_ips = get_recent_ips(user_id, hours=24)
    for ip in suspicious_ips:
        add_to_blocklist(ip, reason=f"incident:{incident_id}", duration_hours=24)
    actions_taken.append(f"ips_blocked:{len(suspicious_ips)}")

    # 5. Preserve evidence
    snapshot_user_activity(user_id, incident_id)
    actions_taken.append("evidence_preserved")

    audit_log("incident_containment",
        incident_id=incident_id,
        target_user=user_id,
        actor=actor_id,
        actions=actions_taken)

    return actions_taken
```

## Post-Incident Review Template

```python
POST_INCIDENT_TEMPLATE = {
    "incident_id": "",
    "severity": "",                     # P1-P4
    "title": "",
    "timeline": {
        "detected_at": "",
        "acknowledged_at": "",
        "contained_at": "",
        "resolved_at": "",
        "post_mortem_completed_at": "",
    },
    "impact": {
        "customers_affected": 0,
        "data_exposed": False,
        "data_types_exposed": [],
        "service_downtime_minutes": 0,
    },
    "root_cause": "",
    "contributing_factors": [],
    "remediation": {
        "immediate_actions": [],          # What was done to stop the bleeding
        "long_term_fixes": [],            # Systemic improvements
        "prevention_measures": [],        # How to prevent recurrence
    },
    "lessons_learned": [],
    "action_items": [
        # {"owner": "", "task": "", "due_date": "", "status": ""}
    ],
}
```
