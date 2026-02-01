# ITGC Implementation Patterns

## RBAC Middleware with Role Enforcement

### Python

```python
from enum import Enum
from functools import wraps
from datetime import datetime, timezone

class FinancialRole(Enum):
    VIEWER = "viewer"              # Read-only access to reports
    DATA_ENTRY = "data_entry"      # Create journal entries (not approve)
    APPROVER = "approver"          # Approve journal entries (not create)
    RECONCILER = "reconciler"      # Perform reconciliations
    ADMIN = "admin"                # System configuration (not financial data)
    AUDITOR = "auditor"            # Read-only access to all data and logs

# Map permissions -- note segregation of duties built in
ROLE_PERMISSIONS = {
    FinancialRole.VIEWER: {"report:read"},
    FinancialRole.DATA_ENTRY: {"journal:create", "journal:edit_draft", "report:read"},
    FinancialRole.APPROVER: {"journal:approve", "journal:reject", "report:read"},
    FinancialRole.RECONCILER: {"reconciliation:perform", "reconciliation:read", "report:read"},
    FinancialRole.ADMIN: {"system:configure", "user:manage", "report:read"},
    FinancialRole.AUDITOR: {"report:read", "journal:read", "audit_log:read",
                            "reconciliation:read", "user:read"},
}

def require_permission(permission: str):
    """Decorator enforcing RBAC on financial system endpoints."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            user = get_authenticated_user()  # Must return unique user, never shared
            if user is None:
                audit_log("access_denied", detail="unauthenticated request",
                          resource=permission)
                raise AuthenticationError("Authentication required")

            user_permissions = set()
            for role in user.roles:
                user_permissions |= ROLE_PERMISSIONS.get(role, set())

            if permission not in user_permissions:
                audit_log("access_denied", user_id=user.id,
                          detail=f"missing permission: {permission}",
                          resource=permission)
                raise AuthorizationError(f"Permission denied: {permission}")

            audit_log("access_granted", user_id=user.id, resource=permission)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@require_permission("journal:create")
def create_journal_entry(data):
    """Only DATA_ENTRY role can create. APPROVER role approves separately."""
    pass
```

### PHP

```php
class FinancialAccessControl
{
    private const ROLE_PERMISSIONS = [
        'viewer'      => ['report:read'],
        'data_entry'  => ['journal:create', 'journal:edit_draft', 'report:read'],
        'approver'    => ['journal:approve', 'journal:reject', 'report:read'],
        'reconciler'  => ['reconciliation:perform', 'reconciliation:read', 'report:read'],
        'admin'       => ['system:configure', 'user:manage', 'report:read'],
        'auditor'     => ['report:read', 'journal:read', 'audit_log:read',
                          'reconciliation:read', 'user:read'],
    ];

    public function requirePermission(string $permission): void
    {
        $user = $this->getAuthenticatedUser();
        if ($user === null) {
            $this->auditLog('access_denied', ['detail' => 'unauthenticated']);
            throw new AuthenticationException('Authentication required');
        }

        $userPermissions = [];
        foreach ($user->getRoles() as $role) {
            $userPermissions = array_merge(
                $userPermissions,
                self::ROLE_PERMISSIONS[$role] ?? []
            );
        }

        if (!in_array($permission, $userPermissions, true)) {
            $this->auditLog('access_denied', [
                'user_id'    => $user->getId(),
                'permission' => $permission,
            ]);
            throw new AuthorizationException("Permission denied: {$permission}");
        }

        $this->auditLog('access_granted', [
            'user_id'    => $user->getId(),
            'permission' => $permission,
        ]);
    }
}
```

## Periodic Access Review Query (Run Quarterly)

```sql
-- SOX ITGC: Quarterly access review report
-- Generates list of all users with financial system access for management review
SELECT
    u.user_id,
    u.full_name,
    u.email,
    u.department,
    GROUP_CONCAT(ur.role_name) AS assigned_roles,
    u.last_login_at,
    u.created_at,
    u.manager_email,
    CASE
        WHEN u.last_login_at < NOW() - INTERVAL 90 DAY THEN 'INACTIVE - REVIEW'
        WHEN u.termination_date IS NOT NULL THEN 'TERMINATED - REMOVE'
        ELSE 'ACTIVE'
    END AS review_status
FROM users u
JOIN user_roles ur ON u.user_id = ur.user_id
WHERE ur.role_name IN ('data_entry', 'approver', 'reconciler', 'admin')
GROUP BY u.user_id
ORDER BY review_status DESC, u.department;
```

## Automated Deprovisioning Check

```python
def check_terminated_users():
    """
    SOX ITGC: Ensure terminated employees have no active access.
    Run daily. Flag any terminated user with active financial system roles.
    """
    terminated_with_access = db.execute("""
        SELECT u.user_id, u.full_name, u.termination_date, ur.role_name
        FROM users u
        JOIN user_roles ur ON u.user_id = ur.user_id
        WHERE u.termination_date IS NOT NULL
          AND u.termination_date <= CURRENT_DATE
          AND ur.is_active = TRUE
    """).fetchall()

    for user in terminated_with_access:
        # Immediately revoke access
        db.execute(
            "UPDATE user_roles SET is_active = FALSE, revoked_at = NOW(), "
            "revoked_reason = 'automated_termination_check' "
            "WHERE user_id = %s AND is_active = TRUE",
            (user.user_id,)
        )
        audit_log("access_revoked_automated",
                  user_id=user.user_id,
                  detail=f"Terminated {user.termination_date}, role: {user.role_name}")
        alert_security_team(user)
```

## Branch Protection Configuration (Generic Git Platform)

```yaml
# Branch protection rules for financial system repositories
main_branch_protection:
  require_pull_request: true
  required_approvals: 2                # At least two reviewers
  dismiss_stale_reviews: true          # Re-review after new commits
  require_code_owner_review: true      # Finance module owners must approve
  require_status_checks:
    - unit-tests
    - integration-tests
    - security-scan
    - sox-audit-trail-check            # Verify audit logging present
  restrict_push: true                  # No direct pushes to main
  require_linear_history: true         # No merge commits -- clean audit trail
  require_signed_commits: true         # Cryptographic author verification
```

## CODEOWNERS Pattern for Financial Modules

```
# Financial modules require finance-team review
/src/ledger/           @finance-team @security-team
/src/journal/          @finance-team @security-team
/src/reconciliation/   @finance-team @security-team
/src/reporting/        @finance-team @security-team
/db/migrations/        @finance-team @dba-team
/config/financial/     @finance-team @ops-team
```

## Pull Request Template with SOX Traceability

```markdown
## Change Request
- Ticket/CR Number: [REQUIRED -- links change to authorized request]
- Business Justification: [Why is this change needed?]
- Financial Impact: [Which financial reports/processes does this affect?]

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] UAT sign-off obtained (attach evidence)
- [ ] Audit trail verified (all financial mutations logged)
- [ ] No direct database modifications (all changes through application layer)

## Segregation of Duties
- [ ] Author is NOT the sole approver
- [ ] Code reviewer is different from the author
- [ ] Deployment will be executed by CI/CD pipeline (not manually)

## Rollback Plan
- Rollback procedure: [describe]
- Data recovery steps: [if applicable]
```

## Deployment Gate Pattern

```python
def pre_deployment_check(deployment):
    """
    SOX Change Management: Automated gate before production deployment.
    Blocks deployment if any control is not satisfied.
    """
    checks = {
        "change_request_linked": deployment.ticket_id is not None,
        "change_request_approved": is_ticket_approved(deployment.ticket_id),
        "approver_not_author": (
            deployment.approved_by != deployment.committed_by
        ),
        "tests_passed": deployment.test_results.all_passed,
        "code_review_approved": (
            deployment.review_approvals >= 2
            and deployment.author not in deployment.reviewers
        ),
        "security_scan_clean": deployment.security_scan.critical_count == 0,
    }

    failed = [name for name, passed in checks.items() if not passed]
    if failed:
        audit_log("deployment_blocked",
                  deployment_id=deployment.id,
                  failed_checks=failed)
        raise DeploymentBlockedError(
            f"SOX controls not satisfied: {', '.join(failed)}"
        )

    audit_log("deployment_approved",
              deployment_id=deployment.id,
              ticket=deployment.ticket_id,
              author=deployment.committed_by,
              approver=deployment.approved_by)
```

## Test Documentation Pattern (Domain 3: Program Development)

```python
class TestJournalEntryPosting:
    """
    SOX Control: JE-001 -- Journal Entry Posting Accuracy
    Control Owner: Finance Controller
    Test Frequency: Every release (automated), quarterly (manual UAT)

    Validates that journal entries are posted accurately to the general ledger
    with correct debits, credits, and account mappings.
    """

    def test_balanced_entry_posts_successfully(self):
        """Debit and credit must balance before posting is allowed."""
        entry = JournalEntry(
            lines=[
                JournalLine(account="4100", debit=Decimal("1000.00")),
                JournalLine(account="1200", credit=Decimal("1000.00")),
            ]
        )
        result = post_journal_entry(entry)
        assert result.status == "posted"
        assert result.gl_impact.total_debits == result.gl_impact.total_credits

    def test_unbalanced_entry_rejected(self):
        """SOX Control: Prevent posting of unbalanced entries."""
        entry = JournalEntry(
            lines=[
                JournalLine(account="4100", debit=Decimal("1000.00")),
                JournalLine(account="1200", credit=Decimal("999.99")),
            ]
        )
        with pytest.raises(UnbalancedEntryError):
            post_journal_entry(entry)

    def test_posting_creates_audit_trail(self):
        """SOX Section 802: Every posting must generate an immutable audit record."""
        entry = create_and_post_valid_entry()
        logs = get_audit_logs(entity_type="journal_entry", entity_id=entry.id)
        assert len(logs) >= 1
        assert logs[0].action == "journal_entry_posted"
        assert logs[0].user_id is not None
        assert logs[0].before_state is not None
        assert logs[0].after_state is not None
```

## Backup Verification Pattern (Domain 4: Computer Operations)

```python
def verify_financial_backup():
    """
    SOX ITGC: Verify backup integrity for financial databases.
    Run daily after backup. Test full restore quarterly.
    """
    backup = get_latest_backup("financial_db")

    # Verify backup exists and is recent
    assert backup is not None, "No backup found"
    age_hours = (datetime.now(timezone.utc) - backup.created_at).total_seconds() / 3600
    assert age_hours < 25, f"Backup is {age_hours:.1f} hours old (max 24)"

    # Verify checksum integrity
    computed_checksum = compute_sha256(backup.file_path)
    assert computed_checksum == backup.stored_checksum, "Backup checksum mismatch"

    # Verify backup size is reasonable (not empty or truncated)
    assert backup.size_bytes > backup.minimum_expected_size, "Backup appears truncated"

    audit_log("backup_verified",
              backup_id=backup.id,
              size=backup.size_bytes,
              checksum=computed_checksum,
              age_hours=round(age_hours, 1))
```

## Batch Job Monitoring

```python
def monitor_financial_batch_jobs():
    """
    SOX ITGC: Monitor critical financial batch processes.
    Alert on failure, late start, or unexpected duration.
    """
    critical_jobs = [
        {"name": "gl_posting_batch",      "max_duration_min": 60,  "deadline": "06:00"},
        {"name": "sub_ledger_sync",       "max_duration_min": 30,  "deadline": "07:00"},
        {"name": "reconciliation_batch",  "max_duration_min": 120, "deadline": "08:00"},
        {"name": "financial_report_gen",  "max_duration_min": 45,  "deadline": "09:00"},
    ]

    for job in critical_jobs:
        last_run = get_last_job_run(job["name"])
        if last_run is None or last_run.status == "failed":
            alert_operations(f"Critical job {job['name']} failed or missing",
                             severity="high")
            audit_log("batch_job_failure", job_name=job["name"],
                      status=last_run.status if last_run else "not_found")
        elif last_run.duration_minutes > job["max_duration_min"]:
            alert_operations(f"Job {job['name']} exceeded duration limit",
                             severity="medium")
```

## Segregation of Duties Enforcement

```python
class SegregationOfDuties:
    """
    SOX Section 404: Enforce segregation of duties on financial transactions.
    No user may perform conflicting duties on the same transaction.
    """

    CONFLICTING_ACTIONS = {
        "journal:create":  {"journal:approve", "journal:post"},
        "journal:approve": {"journal:create"},
        "payment:request": {"payment:approve", "payment:execute"},
        "payment:approve": {"payment:request", "payment:execute"},
        "reconciliation:perform": {"journal:create", "journal:approve"},
    }

    @staticmethod
    def check(user_id: str, action: str, transaction_id: str):
        """
        Before allowing an action, verify the user has not performed
        a conflicting action on the same transaction.
        """
        conflicting = SegregationOfDuties.CONFLICTING_ACTIONS.get(action, set())
        if not conflicting:
            return

        prior_actions = db.execute("""
            SELECT action FROM audit_log
            WHERE transaction_id = %s AND user_id = %s AND action IN %s
        """, (transaction_id, user_id, tuple(conflicting))).fetchall()

        if prior_actions:
            conflicting_action = prior_actions[0].action
            audit_log("sod_violation_blocked",
                      user_id=user_id,
                      attempted_action=action,
                      conflicting_action=conflicting_action,
                      transaction_id=transaction_id)
            raise SoDViolationError(
                f"Segregation of duties violation: user {user_id} cannot perform "
                f"'{action}' after performing '{conflicting_action}' on the same "
                f"transaction {transaction_id}"
            )
```

## CI/CD Pipeline SoD Configuration

```yaml
# CI/CD pipeline configuration enforcing SoD
deployment_pipeline:
  stages:
    - name: build
      trigger: pull_request_merged

    - name: deploy_staging
      trigger: build_success
      # Automated -- no human in loop

    - name: deploy_production
      trigger: manual_approval
      constraints:
        # The person approving production deploy CANNOT be the PR author
        approver_cannot_be:
          - pull_request_author
          - last_commit_author
        required_approvers: 1
        allowed_roles:
          - release_manager
          - ops_lead
```

## Git Branch Protection Enforcing Review Separation

```
# CODEOWNERS: enforce that financial code is reviewed by non-authors
# The CI system verifies the reviewer is not the commit author

/src/financial/**    @finance-reviewers
/src/ledger/**       @finance-reviewers
/src/reporting/**    @finance-reviewers @audit-reviewers
```
