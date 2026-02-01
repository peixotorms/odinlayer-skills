# Data Integrity, Change Management, and Backup Patterns

## Data Integrity: Input Validation at Every Entry Point

```python
class FinancialDataValidator:
    """SOX Section 404: Validate all financial data at system boundaries."""

    @staticmethod
    def validate_amount(value, field_name: str = "amount") -> Decimal:
        """Validate monetary amounts."""
        if value is None:
            raise ValidationError(f"{field_name} is required")
        try:
            amount = Decimal(str(value))
        except (InvalidOperation, ValueError):
            raise ValidationError(f"{field_name} must be a valid number")

        if amount.as_tuple().exponent < -2:
            raise ValidationError(f"{field_name} cannot have more than 2 decimal places")
        if amount < 0:
            raise ValidationError(f"{field_name} cannot be negative")

        return amount

    @staticmethod
    def validate_account_code(code: str) -> str:
        """Validate account codes against chart of accounts."""
        if not code or not code.strip():
            raise ValidationError("Account code is required")
        if not account_exists(code):
            raise ValidationError(f"Account {code} does not exist in chart of accounts")
        if is_account_inactive(code):
            raise ValidationError(f"Account {code} is inactive")
        return code
```

## Data Integrity: Transfer Checksum Verification

```python
def transfer_financial_data(source_system: str, target_system: str,
                            data: list, user_id: str):
    """
    SOX Section 404: Verify data integrity when transferring between systems.
    Checksum ensures no records are lost, added, or modified in transit.
    """
    # Compute source checksum
    record_count = len(data)
    control_total = sum(Decimal(str(r["amount"])) for r in data)
    hash_digest = hashlib.sha256(
        json.dumps(data, sort_keys=True, default=str).encode()
    ).hexdigest()

    transfer_manifest = {
        "source_system": source_system,
        "target_system": target_system,
        "record_count": record_count,
        "control_total": str(control_total),
        "hash_digest": hash_digest,
        "transfer_time": datetime.now(timezone.utc).isoformat(),
    }

    # Send data to target system
    result = send_to_target(target_system, data, transfer_manifest)

    # Verify target received correctly
    if result["received_count"] != record_count:
        raise DataIntegrityError(
            f"Record count mismatch: sent {record_count}, "
            f"received {result['received_count']}"
        )
    if result["received_hash"] != hash_digest:
        raise DataIntegrityError("Data hash mismatch -- data corrupted in transit")

    audit_log("data_transfer_completed",
              entity_type="data_transfer",
              entity_id=f"{source_system}_to_{target_system}",
              user_id=user_id,
              after_state=transfer_manifest)
```

## Data Integrity: Encryption for Financial Data at Rest

```python
from cryptography.fernet import Fernet

class FinancialDataEncryption:
    """
    SOX Section 404: Encrypt sensitive financial data at rest.
    Key management must follow least-privilege and rotation policies.
    """

    def __init__(self, key_provider):
        self.key_provider = key_provider

    def encrypt_field(self, plaintext: str, field_name: str) -> str:
        """Encrypt a single field value. Key is retrieved per-operation."""
        key = self.key_provider.get_current_key()
        f = Fernet(key)
        return f.encrypt(plaintext.encode()).decode()

    def decrypt_field(self, ciphertext: str, field_name: str,
                      user_id: str) -> str:
        """Decrypt a field. Access is logged for audit trail."""
        audit_log("sensitive_field_accessed",
                  entity_type="field_access",
                  entity_id=field_name,
                  user_id=user_id)

        key = self.key_provider.get_current_key()
        f = Fernet(key)
        return f.decrypt(ciphertext.encode()).decode()
```

## Change Management: Ten-Step Pipeline

```
1. Change Request     --> Documented in ticketing system with business justification
2. Impact Assessment  --> Which financial reports/processes affected?
3. Approval           --> Business owner approves (NOT the developer)
4. Development        --> Isolated environment, no production data
5. Testing            --> Unit + integration + security + UAT
6. Code Review        --> By peer (NOT the author)
7. Deploy Approval    --> Separate from dev approval (SoD)
8. Automated Deploy   --> CI/CD pipeline (not manual)
9. Post-Deploy Verify --> Smoke tests + reconciliation check
10. Audit Trail       --> Every step logged with who/what/when
```

## Change Management: CI/CD Pipeline Gate Configuration

```yaml
# Generic CI/CD pipeline for SOX-compliant deployment
pipeline:
  stages:
    - name: validate
      steps:
        - verify_ticket_linked        # Every change tied to approved ticket
        - verify_ticket_approved       # Ticket approved by authorized person
        - verify_branch_protection     # PR approved, reviews not stale

    - name: test
      steps:
        - run_unit_tests
        - run_integration_tests
        - run_security_checks
        - verify_audit_logging         # Custom: ensure financial mutations are logged
        - generate_test_report         # Evidence retained for auditors

    - name: review_gate
      steps:
        - verify_code_review_count: 2  # Minimum 2 approvals
        - verify_author_not_reviewer   # SoD enforcement
        - verify_codeowner_approved    # Finance module owners signed off

    - name: deploy_staging
      steps:
        - deploy_to_staging
        - run_smoke_tests
        - run_reconciliation_check     # Verify financial data integrity

    - name: deploy_production
      requires_manual_approval: true
      approval_constraints:
        approver_cannot_be_author: true
        required_role: release_manager
      steps:
        - deploy_to_production
        - run_smoke_tests
        - run_reconciliation_check
        - notify_change_management     # Close the change ticket
```

## Change Management: Deployment Checklist (Automated Verification)

```python
class SOXDeploymentChecklist:
    """
    SOX Section 404: Automated pre-deployment verification.
    All items must pass before production deployment proceeds.
    """

    def run_all_checks(self, deployment) -> dict:
        results = {}

        # 1. Change request traceability
        results["ticket_linked"] = deployment.ticket_id is not None
        results["ticket_approved"] = self._check_ticket_approved(deployment.ticket_id)

        # 2. Segregation of duties
        results["author_not_deployer"] = (
            deployment.code_author != deployment.deploy_approver
        )
        results["reviewer_not_author"] = (
            deployment.code_author not in deployment.code_reviewers
        )

        # 3. Testing evidence
        results["unit_tests_passed"] = deployment.test_results.unit.passed
        results["integration_tests_passed"] = deployment.test_results.integration.passed
        results["security_scan_clean"] = (
            deployment.security_results.critical == 0
            and deployment.security_results.high == 0
        )

        # 4. Code review
        results["min_reviews_met"] = len(deployment.code_reviewers) >= 2
        results["codeowner_approved"] = deployment.codeowner_approved

        # 5. Environment segregation
        results["staging_tested"] = deployment.staging_smoke_passed
        results["no_prod_data_in_dev"] = self._verify_no_prod_data(deployment)

        # Log result
        all_passed = all(results.values())
        audit_log(
            "deployment_checklist_completed",
            entity_type="deployment",
            entity_id=deployment.id,
            user_id=deployment.deploy_approver,
            after_state={
                "results": results,
                "all_passed": all_passed,
                "ticket_id": deployment.ticket_id,
            }
        )

        if not all_passed:
            failed = [k for k, v in results.items() if not v]
            raise DeploymentBlockedError(
                f"SOX deployment checks failed: {', '.join(failed)}"
            )

        return results
```

## Backup and Recovery: Compliance Verification

```python
class FinancialBackupPolicy:
    """
    SOX ITGC Domain 4: Backup and recovery requirements
    for financial systems.
    """

    REQUIREMENTS = {
        "financial_database": {
            "frequency": "daily",
            "retention_days": 2555,          # 7 years (SOX Section 802)
            "encryption": True,
            "offsite_copy": True,
            "test_restore_frequency": "quarterly",
            "rto_hours": 4,                  # Recovery Time Objective
            "rpo_hours": 1,                  # Recovery Point Objective
        },
        "audit_log_database": {
            "frequency": "daily",
            "retention_days": 2555,          # 7 years -- never shorter than data
            "encryption": True,
            "offsite_copy": True,
            "test_restore_frequency": "quarterly",
            "rto_hours": 4,
            "rpo_hours": 1,
        },
        "document_storage": {
            "frequency": "daily",
            "retention_days": 2555,
            "encryption": True,
            "offsite_copy": True,
            "test_restore_frequency": "annually",
            "rto_hours": 24,
            "rpo_hours": 24,
        },
    }

    def verify_backup_compliance(self, system_name: str):
        """Verify a system's backup meets SOX requirements."""
        policy = self.REQUIREMENTS.get(system_name)
        if not policy:
            raise ValueError(f"No backup policy defined for {system_name}")

        backup = get_latest_backup(system_name)
        issues = []

        if backup is None:
            issues.append("No backup found")
        else:
            # Check age
            age_hours = (datetime.now(timezone.utc) - backup.timestamp).total_seconds() / 3600
            max_age = 25 if policy["frequency"] == "daily" else 169  # daily or weekly
            if age_hours > max_age:
                issues.append(f"Backup is {age_hours:.0f} hours old (max {max_age})")

            # Check encryption
            if policy["encryption"] and not backup.is_encrypted:
                issues.append("Backup is not encrypted")

            # Check offsite copy
            if policy["offsite_copy"] and not backup.has_offsite_copy:
                issues.append("No offsite copy found")

            # Check integrity
            if not verify_backup_checksum(backup):
                issues.append("Backup checksum verification failed")

        # Check test restore currency
        last_test = get_last_restore_test(system_name)
        if last_test is None:
            issues.append("No restore test on record")
        else:
            test_age_days = (date.today() - last_test.date).days
            max_days = 90 if policy["test_restore_frequency"] == "quarterly" else 365
            if test_age_days > max_days:
                issues.append(f"Last restore test was {test_age_days} days ago (max {max_days})")

        audit_log("backup_compliance_check",
                  entity_type="backup", entity_id=system_name,
                  user_id="system_batch",
                  after_state={"issues": issues, "compliant": len(issues) == 0})

        return {"system": system_name, "compliant": len(issues) == 0, "issues": issues}
```
