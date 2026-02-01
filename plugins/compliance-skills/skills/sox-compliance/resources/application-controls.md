# Application Controls Implementation Patterns

## Input Controls: Journal Entry Validator

```python
from decimal import Decimal, InvalidOperation
from datetime import date

class JournalEntryValidator:
    """SOX Section 404: Input controls for journal entries."""

    MAX_LINE_AMOUNT = Decimal("999999999.99")  # Configurable threshold
    VALID_CURRENCIES = {"USD", "EUR", "GBP", "JPY", "CAD"}

    def validate(self, entry: dict) -> list:
        errors = []

        # Completeness: all required fields present
        required = ["date", "description", "lines", "created_by"]
        for field in required:
            if not entry.get(field):
                errors.append(f"Missing required field: {field}")

        # Date validation: no future-dated entries without approval
        if entry.get("date"):
            entry_date = self._parse_date(entry["date"])
            if entry_date and entry_date > date.today():
                errors.append("Future-dated entry requires supervisory approval")

            # No entries in closed periods
            if entry_date and self._is_period_closed(entry_date):
                errors.append(f"Period {entry_date.strftime('%Y-%m')} is closed")

        # Line validation
        if entry.get("lines"):
            total_debit = Decimal("0")
            total_credit = Decimal("0")

            for i, line in enumerate(entry["lines"]):
                # Amount validation
                try:
                    debit = Decimal(str(line.get("debit", "0")))
                    credit = Decimal(str(line.get("credit", "0")))
                except InvalidOperation:
                    errors.append(f"Line {i+1}: Invalid amount format")
                    continue

                if debit < 0 or credit < 0:
                    errors.append(f"Line {i+1}: Negative amounts not allowed")
                if debit > self.MAX_LINE_AMOUNT or credit > self.MAX_LINE_AMOUNT:
                    errors.append(f"Line {i+1}: Amount exceeds threshold")
                if debit > 0 and credit > 0:
                    errors.append(f"Line {i+1}: Line cannot have both debit and credit")

                # Account validation
                if not self._is_valid_account(line.get("account")):
                    errors.append(f"Line {i+1}: Invalid account code")

                total_debit += debit
                total_credit += credit

            # Balance check: debits must equal credits
            if total_debit != total_credit:
                errors.append(
                    f"Entry is unbalanced: debits={total_debit}, credits={total_credit}"
                )

        # Duplicate detection
        if self._is_duplicate(entry):
            errors.append("Potential duplicate entry detected -- requires override approval")

        return errors
```

## Approval Workflow with SoD Enforcement

```python
class JournalApprovalWorkflow:
    """
    SOX Section 404: Journal entries require approval before posting.
    The approver MUST be different from the creator (SoD).
    """

    APPROVAL_THRESHOLDS = [
        # (amount_threshold, required_approval_level)
        (Decimal("10000"),    "supervisor"),
        (Decimal("100000"),   "controller"),
        (Decimal("1000000"),  "cfo"),
    ]

    def submit_for_approval(self, entry_id: str, submitter_id: str):
        entry = self.get_entry(entry_id)

        if entry.created_by != submitter_id:
            raise PermissionError("Only the creator can submit for approval")

        max_amount = max(
            sum(line.debit for line in entry.lines),
            sum(line.credit for line in entry.lines)
        )

        required_level = "supervisor"  # Default
        for threshold, level in self.APPROVAL_THRESHOLDS:
            if max_amount >= threshold:
                required_level = level

        entry.status = "pending_approval"
        entry.required_approval_level = required_level
        self.save(entry)

        audit_log("journal_submitted_for_approval",
                  entity_type="journal_entry", entity_id=entry_id,
                  user_id=submitter_id,
                  after_state={"status": "pending_approval",
                               "required_level": required_level})

    def approve(self, entry_id: str, approver_id: str):
        entry = self.get_entry(entry_id)

        # SOX SoD: Approver cannot be the creator
        if entry.created_by == approver_id:
            audit_log("sod_violation_blocked", user_id=approver_id,
                      entity_type="journal_entry", entity_id=entry_id,
                      after_state={"violation": "creator_cannot_approve"})
            raise SoDViolationError("Creator cannot approve their own entry")

        # Verify approver has sufficient authority level
        if not self.user_has_level(approver_id, entry.required_approval_level):
            raise PermissionError(
                f"Approver must have {entry.required_approval_level} level"
            )

        before = {"status": entry.status}
        entry.status = "approved"
        entry.approved_by = approver_id
        entry.approved_at = datetime.now(timezone.utc)
        self.save(entry)

        audit_log("journal_entry_approved",
                  entity_type="journal_entry", entity_id=entry_id,
                  user_id=approver_id,
                  before_state=before,
                  after_state={"status": "approved",
                               "approved_by": approver_id})
```

## Processing Controls: Transaction Atomicity

```python
def post_journal_entry(entry_id: str, user_id: str):
    """
    SOX Section 404: Post journal entry to general ledger.
    All-or-nothing: either all lines post or none do.
    """
    entry = get_journal_entry(entry_id)

    if entry.status != "approved":
        raise InvalidStateError("Only approved entries can be posted")

    before_balances = {}
    try:
        db.begin_transaction()

        for line in entry.lines:
            # Capture before state for audit trail
            balance = get_account_balance(line.account)
            before_balances[line.account] = balance

            # Update general ledger
            update_gl_balance(
                account=line.account,
                debit=line.debit,
                credit=line.credit,
                entry_id=entry_id,
                period=entry.posting_period,
            )

        # Update entry status
        entry.status = "posted"
        entry.posted_by = user_id
        entry.posted_at = datetime.now(timezone.utc)
        save_entry(entry)

        db.commit()

    except Exception as e:
        db.rollback()
        audit_log("journal_posting_failed",
                  entity_type="journal_entry", entity_id=entry_id,
                  user_id=user_id,
                  after_state={"error": str(e)})
        raise

    # Audit trail for successful posting (after commit)
    after_balances = {acct: get_account_balance(acct) for acct in before_balances}
    audit_log("journal_entry_posted",
              entity_type="journal_entry", entity_id=entry_id,
              user_id=user_id,
              before_state={"balances": before_balances, "status": "approved"},
              after_state={"balances": after_balances, "status": "posted"})
```

## Sub-Ledger to General Ledger Reconciliation

```python
def reconcile_sub_ledger_to_gl(sub_ledger: str, period: str):
    """
    SOX Section 404: Automated reconciliation between sub-ledger and GL.
    Run daily. Discrepancies must be investigated and resolved.
    """
    sub_total = get_sub_ledger_total(sub_ledger, period)
    gl_total = get_gl_balance_for_sub_ledger(sub_ledger, period)

    difference = sub_total - gl_total
    is_reconciled = abs(difference) < Decimal("0.01")  # Penny tolerance

    result = {
        "sub_ledger": sub_ledger,
        "period": period,
        "sub_ledger_total": str(sub_total),
        "gl_total": str(gl_total),
        "difference": str(difference),
        "reconciled": is_reconciled,
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }

    audit_log("reconciliation_performed",
              entity_type="reconciliation",
              entity_id=f"{sub_ledger}_{period}",
              user_id="system_batch",
              after_state=result)

    if not is_reconciled:
        alert_finance_team(
            f"Reconciliation discrepancy: {sub_ledger} for {period} "
            f"differs by {difference}"
        )
        create_reconciliation_item(sub_ledger, period, difference)

    return result
```

## Output Controls: Report Authorization

```python
def generate_financial_report(report_type: str, period: str, user_id: str):
    """
    SOX Section 302/404: Generate financial report with output controls.
    Only authorized recipients receive reports. All access is logged.
    """
    # Verify user is authorized to receive this report
    if not is_authorized_report_recipient(user_id, report_type):
        audit_log("report_access_denied", user_id=user_id,
                  entity_type="report", entity_id=f"{report_type}_{period}")
        raise AuthorizationError("Not authorized for this report")

    report_data = compile_report(report_type, period)

    # Output-to-input reconciliation: verify report totals match source
    source_totals = get_source_totals(report_type, period)
    report_totals = extract_report_totals(report_data)

    if source_totals != report_totals:
        audit_log("report_reconciliation_failed",
                  entity_type="report", entity_id=f"{report_type}_{period}",
                  user_id="system",
                  after_state={"source": str(source_totals),
                               "report": str(report_totals)})
        raise DataIntegrityError("Report totals do not match source data")

    # Completeness check
    expected_accounts = get_expected_accounts(report_type)
    reported_accounts = extract_accounts(report_data)
    missing = expected_accounts - reported_accounts
    if missing:
        raise DataIntegrityError(f"Report missing accounts: {missing}")

    # Log successful report generation
    audit_log("report_generated",
              entity_type="report", entity_id=f"{report_type}_{period}",
              user_id=user_id,
              after_state={"type": report_type, "period": period,
                           "total": str(report_totals)})

    return report_data
```
