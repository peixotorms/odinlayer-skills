# Availability & Processing Integrity Controls — Implementation Code

## Circuit Breaker Pattern

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"           # Normal operation
    OPEN = "open"               # Failing — reject requests immediately
    HALF_OPEN = "half_open"     # Testing if service recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30,
                 success_threshold=3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                audit_log("circuit_breaker_rejected",
                          service=func.__name__, state="open")
                raise ServiceUnavailableError("Circuit breaker is open")

        try:
            result = func(*args, **kwargs)
            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.success_threshold:
                    self.state = CircuitState.CLOSED
                    self.failure_count = 0
                    audit_log("circuit_breaker_closed", service=func.__name__)
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
                audit_log("circuit_breaker_opened",
                          service=func.__name__, failures=self.failure_count)
            raise
```

## Retry with Exponential Backoff

```javascript
async function retryWithBackoff(fn, options = {}) {
  const {
    maxRetries = 3,
    baseDelayMs = 1000,
    maxDelayMs = 30000,
    retryableErrors = [502, 503, 504, "ECONNRESET", "ETIMEDOUT"],
  } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const isRetryable = retryableErrors.some(
        (e) => error.status === e || error.code === e
      );
      if (!isRetryable || attempt === maxRetries) {
        throw error;
      }
      const delay = Math.min(
        baseDelayMs * Math.pow(2, attempt) + Math.random() * 1000,
        maxDelayMs
      );
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}
```

## Backup Verification

```python
def verify_backup(backup_id):
    """
    CC7: Regularly test backup restoration.
    Run monthly; log results as SOC 2 evidence.
    """
    result = {
        "backup_id": backup_id,
        "test_date": utcnow().isoformat(),
        "steps": [],
    }

    # 1. Verify backup exists and is not corrupted
    backup = get_backup(backup_id)
    checksum_valid = verify_checksum(backup)
    result["steps"].append({
        "step": "checksum_verification",
        "passed": checksum_valid,
    })

    # 2. Restore to isolated test environment
    test_env = create_isolated_restore_environment()
    restore_success = restore_backup(backup, test_env)
    result["steps"].append({
        "step": "restore_to_test",
        "passed": restore_success,
    })

    # 3. Verify data integrity after restore
    record_count_match = verify_record_counts(test_env)
    sample_data_valid = verify_sample_records(test_env)
    result["steps"].append({
        "step": "data_integrity",
        "passed": record_count_match and sample_data_valid,
    })

    # 4. Clean up
    destroy_environment(test_env)

    result["overall_passed"] = all(s["passed"] for s in result["steps"])
    audit_log("backup_verification", **result)
    return result
```

## Input Validation (Python)

```python
from pydantic import BaseModel, validator, constr
from typing import Optional

class CreateCustomerRequest(BaseModel):
    """Validate all inputs at API boundary. Reject invalid data early."""
    name: constr(min_length=1, max_length=255, strip_whitespace=True)
    email: constr(regex=r"^[^@\s]+@[^@\s]+\.[^@\s]+$", max_length=254)
    plan: str
    metadata: Optional[dict] = None

    @validator("plan")
    def validate_plan(cls, v):
        allowed = ["free", "starter", "professional", "enterprise"]
        if v not in allowed:
            raise ValueError(f"Plan must be one of: {allowed}")
        return v

    @validator("metadata")
    def validate_metadata(cls, v):
        if v and len(str(v)) > 10000:
            raise ValueError("Metadata too large")
        return v
```

## Input Validation (Node.js)

```javascript
// Validate at every API boundary
function validateCreateCustomer(body) {
  const errors = [];

  if (!body.name || typeof body.name !== "string") {
    errors.push("name is required and must be a string");
  } else if (body.name.length > 255) {
    errors.push("name must be 255 characters or fewer");
  }

  if (!body.email || !/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(body.email)) {
    errors.push("valid email is required");
  }

  const allowedPlans = ["free", "starter", "professional", "enterprise"];
  if (!allowedPlans.includes(body.plan)) {
    errors.push(`plan must be one of: ${allowedPlans.join(", ")}`);
  }

  if (errors.length > 0) {
    auditLog("validation_failed", {
      endpoint: "/customers",
      errors,
      ip: req.ip,
    });
    throw new ValidationError(errors);
  }
}
```

## Idempotency

```python
def process_payment(idempotency_key, payment_data):
    """
    Idempotency ensures processing integrity — the same request
    produces the same result regardless of how many times it is sent.
    """
    # Check for existing result with this key
    existing = db.query(
        "SELECT result FROM idempotency_keys WHERE key = %s AND expires_at > NOW()",
        [idempotency_key]
    )
    if existing:
        audit_log("idempotent_replay", key=idempotency_key)
        return existing.result

    # Process the payment
    result = execute_payment(payment_data)

    # Store result for future duplicate requests
    db.execute(
        """INSERT INTO idempotency_keys (key, result, created_at, expires_at)
           VALUES (%s, %s, NOW(), NOW() + INTERVAL '24 hours')""",
        [idempotency_key, serialize(result)]
    )

    audit_log("payment_processed", key=idempotency_key,
              amount=payment_data["amount"], outcome="success")
    return result
```

## Data Reconciliation

```python
def reconcile_billing(period_start, period_end):
    """
    Processing integrity check: reconcile usage records
    against invoices to detect discrepancies.
    """
    usage_records = get_usage_records(period_start, period_end)
    invoices = get_invoices(period_start, period_end)

    discrepancies = []
    for customer_id in set(r.customer_id for r in usage_records):
        usage_total = sum(r.amount for r in usage_records
                         if r.customer_id == customer_id)
        invoice_total = sum(i.amount for i in invoices
                           if i.customer_id == customer_id)

        if abs(usage_total - invoice_total) > 0.01:
            discrepancies.append({
                "customer_id": customer_id,
                "usage_total": usage_total,
                "invoice_total": invoice_total,
                "difference": usage_total - invoice_total,
            })

    result = {
        "period": f"{period_start} to {period_end}",
        "records_checked": len(usage_records),
        "invoices_checked": len(invoices),
        "discrepancies_found": len(discrepancies),
        "discrepancies": discrepancies,
        "status": "pass" if not discrepancies else "fail",
    }

    audit_log("billing_reconciliation", **result)
    if discrepancies:
        send_alert("billing_discrepancy", result)
    return result
```
