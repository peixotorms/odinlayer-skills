# Technical Safeguards Code Examples

Implementation examples for HIPAA Security Rule technical safeguards (45 CFR 164.312) and administrative safeguards (45 CFR 164.308).

## 4.1 Access Control — 164.312(a)(1)

### Unique User Identification (REQUIRED)

No shared accounts, every action tied to a unique user:

```python
# CORRECT — unique user identity on every ePHI access
class PHIAccessMiddleware:
    def process_request(self, request):
        if not request.user or not request.user.is_authenticated:
            raise PermissionDenied("Authentication required for ePHI access")
        request.phi_access_context = {
            "user_id": request.user.id,
            "session_id": request.session.session_key,
            "ip_address": request.META.get("REMOTE_ADDR"),
            "timestamp": datetime.utcnow().isoformat(),
        }

# WRONG — shared credentials
# admin_user = "clinic_admin"
# admin_pass = "shared_password_2024"
```

```php
function validatePHIAccess(int $userId, string $resource): bool {
    if ($userId <= 0) {
        throw new AccessDeniedException('Anonymous access to ePHI prohibited');
    }
    $user = User::find($userId);
    if ($user->is_service_account && !$user->has_phi_authorization) {
        throw new AccessDeniedException('Service account lacks ePHI authorization');
    }
    return RBACPolicy::check($userId, $resource, 'read');
}
```

### Emergency Access Procedure (REQUIRED)

"Break glass" mechanism with full audit:

```python
def emergency_phi_access(requesting_user_id, patient_id, reason):
    """Bypasses normal RBAC but logs extensively and notifies compliance."""
    if not reason or len(reason) < 20:
        raise ValueError("Emergency access requires detailed justification")
    audit_log.critical("EMERGENCY_PHI_ACCESS", {
        "user_id": requesting_user_id, "patient_id": patient_id,
        "reason": reason, "requires_review": True,
    })
    notify_compliance_team(event="emergency_phi_access", user_id=requesting_user_id)
    grant_temporary_access(user_id=requesting_user_id, patient_id=patient_id,
                           duration_hours=4, access_level="read_only")
```

### Automatic Logoff (ADDRESSABLE)

Session timeout for unattended workstations:

```javascript
const PHI_SESSION_TIMEOUT_MS = 15 * 60 * 1000; // 15 minutes
function phiSessionMiddleware(req, res, next) {
  if (!req.session?.lastActivity) {
    return res.status(401).json({ error: "Session expired" });
  }
  if (Date.now() - req.session.lastActivity > PHI_SESSION_TIMEOUT_MS) {
    auditLog("SESSION_TIMEOUT", { userId: req.session.userId });
    req.session.destroy();
    return res.status(401).json({ error: "Session expired due to inactivity" });
  }
  req.session.lastActivity = Date.now();
  next();
}
```

### Encryption and Decryption (ADDRESSABLE)

AES-256 for ePHI at rest:

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

class PHIEncryption:
    """AES-256-GCM encryption for ePHI fields at rest."""
    def __init__(self, key: bytes):
        if len(key) != 32:
            raise ValueError("AES-256 requires a 32-byte key")
        self._gcm = AESGCM(key)

    def encrypt(self, plaintext: str) -> bytes:
        nonce = os.urandom(12)
        ciphertext = self._gcm.encrypt(nonce, plaintext.encode("utf-8"), None)
        return nonce + ciphertext

    def decrypt(self, data: bytes) -> str:
        return self._gcm.decrypt(data[:12], data[12:], None).decode("utf-8")

phi_crypto = PHIEncryption(key=load_key_from_hsm_or_kms())
encrypted_ssn = phi_crypto.encrypt("123-45-6789")
```

```javascript
const crypto = require("crypto");
function encryptPHI(plaintext, keyBuffer) {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv("aes-256-gcm", keyBuffer, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, "utf8"), cipher.final()]);
  return Buffer.concat([iv, cipher.getAuthTag(), encrypted]);
}
function decryptPHI(buf, keyBuffer) {
  const decipher = crypto.createDecipheriv("aes-256-gcm", keyBuffer, buf.subarray(0, 12));
  decipher.setAuthTag(buf.subarray(12, 28));
  return decipher.update(buf.subarray(28), null, "utf8") + decipher.final("utf8");
}
```

## 4.2 Audit Controls — 164.312(b) (REQUIRED)

Log ALL access to ePHI. Retain logs for 6+ years. **What to log:** who, what resource, when, from where, action, outcome. **What NEVER to log:** the PHI content itself.

```python
def phi_audit_log(event_type, user_id, resource_type, resource_id,
                  action, outcome, ip_address, details=None):
    """
    Structured audit log. NEVER include PHI content in details.
    Log resource_id (e.g., patient_id=4821) but NEVER
    resource content (e.g., patient_name="John Smith").
    """
    entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "event_type": event_type,
        "user_id": user_id,
        "resource_type": resource_type,   # "patient_record", "lab_result"
        "resource_id": resource_id,       # numeric ID only
        "action": action,                 # "read", "write", "delete", "export"
        "outcome": outcome,               # "success", "denied", "error"
        "ip_address": ip_address,
        "details": details,               # non-PHI metadata only
    }
    append_to_immutable_log(json.dumps(entry))  # append-only, no UPDATE/DELETE

# CORRECT
phi_audit_log("PHI_READ", 42, "patient_record", 4821, "read", "success",
              "10.0.1.50", details={"fields_accessed": ["dob", "diagnosis"]})
# WRONG — never log PHI content
# phi_audit_log(..., details={"patient_name": "John Smith", "ssn": "123-45-6789"})
```

```php
function phiAuditLog(string $eventType, int $userId, string $resourceType,
                     int $resourceId, string $action, string $outcome, array $details = []): void {
    $prohibited = ['name', 'ssn', 'dob', 'address', 'phone', 'email', 'diagnosis'];
    foreach ($prohibited as $field) {
        if (isset($details[$field])) {
            throw new InvalidArgumentException("PHI field '{$field}' prohibited in audit logs");
        }
    }
    DB::table('phi_audit_log')->insert([
        'timestamp' => gmdate('Y-m-d\TH:i:s\Z'), 'event_type' => $eventType,
        'user_id' => $userId, 'resource_type' => $resourceType,
        'resource_id' => $resourceId, 'action' => $action,
        'outcome' => $outcome, 'details' => json_encode($details),
    ]);
    // Table must have NO UPDATE or DELETE grants for application users
}
```

## 4.3 Integrity Controls — 164.312(c)(1) (REQUIRED)

Protect ePHI from unauthorized alteration. Verify data has not been tampered with.

```python
import hmac, hashlib, json

class PHIIntegrity:
    def __init__(self, key: bytes):
        self._key = key

    def compute_checksum(self, record: dict) -> str:
        canonical = json.dumps(record, sort_keys=True, separators=(",", ":"))
        return hmac.new(self._key, canonical.encode(), hashlib.sha256).hexdigest()

    def verify(self, record: dict, expected: str) -> bool:
        if not hmac.compare_digest(self.compute_checksum(record), expected):
            phi_audit_log("INTEGRITY_VIOLATION", ...)
            raise IntegrityError("ePHI record integrity check failed")
        return True
```

```sql
-- Version tracking prevents unauthorized alteration
CREATE TABLE patient_records (
    id BIGINT PRIMARY KEY,
    version INT NOT NULL DEFAULT 1,
    checksum VARCHAR(64) NOT NULL,
    modified_by INT NOT NULL REFERENCES users(id),
    modified_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- Trigger: reject updates where NEW.version <= OLD.version
```

## 4.4 Person or Entity Authentication — 164.312(d) (REQUIRED)

MFA strongly recommended. Short-lived tokens for ePHI access.

```python
def issue_phi_access_token(user_id, roles, mfa_verified):
    if not mfa_verified:
        raise AuthenticationError("MFA required for ePHI access")
    return jwt.encode({
        "sub": str(user_id), "roles": roles, "mfa": True,
        "exp": datetime.utcnow() + timedelta(minutes=30),
    }, PHI_JWT_SECRET, algorithm="HS256")

def verify_phi_access(token):
    payload = jwt.decode(token, PHI_JWT_SECRET, algorithms=["HS256"])
    if not payload.get("mfa"):
        raise AuthenticationError("Token lacks MFA verification")
    return payload
```

```javascript
// System-to-system auth: signed requests, never API keys in query strings
function signPHIRequest(method, path, body, secretKey) {
  const timestamp = Date.now().toString();
  const signature = crypto.createHmac("sha256", secretKey)
    .update(`${method}\n${path}\n${timestamp}\n${body || ""}`)
    .digest("hex");
  return { "X-PHI-Timestamp": timestamp, "X-PHI-Signature": signature };
}
```

## 4.5 Transmission Security — 164.312(e)(1) (ADDRESSABLE)

TLS 1.2+ mandatory. Never transmit PHI in URLs or query parameters.

```python
def create_phi_ssl_context():
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.minimum_version = ssl.TLSVersion.TLSv1_2
    ctx.set_ciphers("ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:!aNULL:!MD5:!DSS")
    ctx.check_hostname = True
    ctx.verify_mode = ssl.CERT_REQUIRED
    return ctx
```

```javascript
// WRONG: GET /api/patients?ssn=123-45-6789&name=John+Smith
// CORRECT: POST with PHI in encrypted body only
app.post("/api/patients/search", requireHTTPS, phiAuth, (req, res) => {
  const { criteria } = req.body; // PHI in body, never in URL
});

function requireHTTPS(req, res, next) {
  if (!req.secure && req.headers["x-forwarded-proto"] !== "https") {
    auditLog("INSECURE_PHI_ATTEMPT", { path: req.path, ip: req.ip });
    return res.status(403).json({ error: "ePHI endpoints require HTTPS" });
  }
  next();
}
```

## 5. Administrative Safeguards (45 CFR 164.308)

### Access Management — RBAC with Minimum Necessary (164.308(a)(4))

```python
PHI_PERMISSIONS = {
    "receptionist": {"patient_record": ["read:demographics"]},
    "nurse":        {"patient_record": ["read:demographics", "read:vitals", "write:vitals"]},
    "physician":    {"patient_record": ["read:demographics", "read:vitals", "read:diagnosis",
                                        "write:diagnosis", "write:prescriptions"]},
    "billing":      {"patient_record": ["read:demographics", "read:billing"]},
    "researcher":   {"patient_record": ["read:de_identified"]},
}

def check_phi_permission(role, resource, action_scope):
    allowed = PHI_PERMISSIONS.get(role, {}).get(resource, [])
    if action_scope not in allowed:
        phi_audit_log("PHI_ACCESS_DENIED", ...)
        raise PermissionDenied(f"Role '{role}' lacks '{action_scope}' on '{resource}'")
```

### Incident Response — Anomaly Detection (164.308(a)(6))

```python
def check_access_anomaly(user_id, resource_type):
    recent_count = count_phi_accesses(user_id, resource_type, last_hours=1)
    baseline = get_user_baseline(user_id, resource_type)
    if recent_count > baseline * 3:
        alert_security_team(event="PHI_ACCESS_ANOMALY", user_id=user_id,
                            recent_count=recent_count, baseline=baseline)
```

### Contingency Plan — Encrypted Backups (164.308(a)(7))

```python
def backup_phi_database(db_path, backup_path, key):
    raw = create_database_dump(db_path)
    encrypted = phi_encrypt(raw, key)
    checksum = hashlib.sha256(encrypted).hexdigest()
    write_file(backup_path, encrypted)
    write_file(backup_path + ".sha256", checksum)
    verify_backup(backup_path, checksum)
    phi_audit_log("PHI_BACKUP_CREATED", details={"size": len(encrypted), "checksum": checksum})
```

### BAA Error Tracking — PHI Scrubbing

```python
# WRONG — PHI leaks into error tracking without BAA
try:
    process_patient(patient)
except Exception as e:
    sentry_sdk.capture_exception(e)  # sends patient data to Sentry

# CORRECT — scrub PHI before reporting
def before_send(event, hint):
    if "request" in event and "data" in event["request"]:
        event["request"]["data"] = "[REDACTED — may contain PHI]"
    if "exception" in event:
        for exc in event["exception"].get("values", []):
            for frame in exc.get("stacktrace", {}).get("frames", []):
                frame.pop("vars", None)
    return event

sentry_sdk.init(dsn="...", before_send=before_send)
```
