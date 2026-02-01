# Breach Response Code Examples

Implementation examples for HIPAA Breach Notification Rule (45 CFR 164.400-414) — detection, containment, logging, and notification tracking.

## Detection

Monitor for indicators of potential PHI breach: bulk access, off-hours access, new IP addresses.

```python
def detect_potential_breach(user_id, action, resource_count):
    alerts = []
    if resource_count > 100:
        alerts.append({"type": "BULK_ACCESS", "severity": "high"})
    hour = datetime.utcnow().hour
    if hour < 6 or hour > 22:
        alerts.append({"type": "OFF_HOURS_ACCESS", "severity": "medium"})
    if is_new_ip_for_user(user_id):
        alerts.append({"type": "NEW_IP_ACCESS", "severity": "medium"})
    for a in alerts:
        phi_audit_log("BREACH_INDICATOR", user_id=user_id, details=a)
        if a["severity"] == "high":
            notify_security_team(a)
```

## Containment

Immediately revoke access and preserve evidence when a potential breach is detected.

```python
def contain_potential_breach(user_id, reason):
    revoke_all_sessions(user_id)
    revoke_all_tokens(user_id)
    lock_user_account(user_id, reason=reason)
    snapshot_audit_logs(user_id, reason=reason)
    phi_audit_log("BREACH_CONTAINMENT", user_id=user_id,
                  details={"reason": reason, "actions": [
                      "sessions_revoked", "tokens_revoked", "account_locked"]})
```

## Breach Log Schema

Track all breach incidents with notification status for compliance reporting.

```sql
CREATE TABLE breach_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    detected_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    detected_by VARCHAR(100) NOT NULL,
    breach_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    affected_record_count INT,
    affected_record_type VARCHAR(100),
    description TEXT NOT NULL,                  -- no PHI in description
    containment_actions TEXT,
    notification_required BOOLEAN DEFAULT FALSE,
    hhs_notified_at TIMESTAMP NULL,            -- within 60 days
    individuals_notified_at TIMESTAMP NULL,
    media_notified_at TIMESTAMP NULL,           -- required if 500+ records
    resolved_at TIMESTAMP NULL
);
```

## Notification Requirements

- **HHS notification**: Within 60 days of discovery for breaches affecting 500+ individuals. Annual report for smaller breaches.
- **Individual notification**: Written notice to affected individuals within 60 days. Must include: description of breach, types of information involved, steps individuals should take, what the entity is doing, contact information.
- **Media notification**: Required if 500+ individuals in a single state/jurisdiction are affected. Prominent media outlets in the state.

## Mobile and API PHI Protection

### No PHI in Push Notifications

```javascript
// WRONG: body: "Your HIV test result is negative. Dr. Smith reviewed on 01/15."
// CORRECT:
sendPushNotification(deviceToken, {
  title: "New Message",
  body: "You have a new message from your care team. Tap to view.",
  data: { type: "lab_result", requiresAuth: true },
});
```

### No PHI in Client-Side Storage Without Encryption

```javascript
// WRONG: localStorage.setItem("patientData", JSON.stringify(record));
// CORRECT: store reference only, fetch PHI on demand with auth
sessionStorage.setItem("currentPatientId", patientId);
```

### Rate Limiting on PHI Endpoints

```python
PHI_RATE_LIMITS = {
    "patient_read": {"requests": 60, "window_seconds": 60},
    "patient_export": {"requests": 5, "window_seconds": 3600},
}
```
