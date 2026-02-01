# Access Controls & Encryption Standards — Implementation Code

## RBAC Implementation (Python)

```python
from functools import wraps
from enum import Enum

class Permission(Enum):
    READ_DATA = "read:data"
    WRITE_DATA = "write:data"
    MANAGE_USERS = "manage:users"
    ADMIN_SETTINGS = "admin:settings"
    VIEW_AUDIT_LOG = "view:audit_log"

ROLE_PERMISSIONS = {
    "viewer":  [Permission.READ_DATA],
    "editor":  [Permission.READ_DATA, Permission.WRITE_DATA],
    "admin":   [Permission.READ_DATA, Permission.WRITE_DATA, Permission.MANAGE_USERS,
                Permission.VIEW_AUDIT_LOG],
    "owner":   [p for p in Permission],
}

def require_permission(permission: Permission):
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            user = get_authenticated_user(request)
            user_permissions = ROLE_PERMISSIONS.get(user.role, [])
            if permission not in user_permissions:
                audit_log("authorization_denied", user=user.id,
                          resource=request.path, permission=permission.value)
                raise ForbiddenError("Insufficient permissions")
            audit_log("authorization_granted", user=user.id,
                      resource=request.path, permission=permission.value)
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

@require_permission(Permission.MANAGE_USERS)
def create_user(request):
    # Only admin and owner roles reach here
    pass
```

## RBAC Implementation (Node.js)

```javascript
// Express middleware for RBAC
function requirePermission(permission) {
  return (req, res, next) => {
    const user = req.authenticatedUser;
    const userPerms = getRolePermissions(user.role);

    if (!userPerms.includes(permission)) {
      auditLog({
        event: "authorization_denied",
        userId: user.id,
        resource: req.path,
        permission,
        ip: req.ip,
        timestamp: new Date().toISOString(),
      });
      return res.status(403).json({ error: "Insufficient permissions" });
    }

    auditLog({
      event: "authorization_granted",
      userId: user.id,
      resource: req.path,
      permission,
      ip: req.ip,
      timestamp: new Date().toISOString(),
    });
    next();
  };
}

router.post("/users", requirePermission("manage:users"), createUser);
router.get("/data", requirePermission("read:data"), getData);
```

## Session Management

```python
SESSION_CONFIG = {
    "max_idle_timeout_minutes": 15,         # CC6: Auto-logout after inactivity
    "max_absolute_timeout_hours": 12,       # CC6: Force re-auth after 12 hours
    "max_concurrent_sessions": 3,           # CC6: Limit parallel sessions
    "require_mfa_for_sensitive_actions": True,
    "rotate_session_id_on_auth": True,      # Prevent session fixation
    "secure_cookie_flags": {
        "httponly": True,
        "secure": True,
        "samesite": "Strict",
    },
}

def validate_session(session):
    now = datetime.utcnow()
    if now - session.last_activity > timedelta(minutes=15):
        terminate_session(session.id, reason="idle_timeout")
        audit_log("session_expired", session_id=session.id, reason="idle_timeout")
        raise SessionExpiredError()
    if now - session.created_at > timedelta(hours=12):
        terminate_session(session.id, reason="absolute_timeout")
        audit_log("session_expired", session_id=session.id, reason="absolute_timeout")
        raise SessionExpiredError()
    session.last_activity = now
```

## API Key Management

```python
def create_api_key(user_id, scopes, expires_days=90):
    """Create scoped, expiring API key with audit trail."""
    key_id = generate_uuid()
    raw_key = generate_secure_random(32)
    hashed_key = hash_with_salt(raw_key)

    store_api_key({
        "key_id": key_id,
        "hashed_key": hashed_key,
        "user_id": user_id,
        "scopes": scopes,               # Least privilege: only needed permissions
        "created_at": utcnow(),
        "expires_at": utcnow() + timedelta(days=expires_days),
        "last_used_at": None,
        "last_rotated_at": utcnow(),
    })
    audit_log("api_key_created", user_id=user_id, key_id=key_id, scopes=scopes)
    return {"key_id": key_id, "key": raw_key}  # Raw key shown once only

def validate_api_key(raw_key):
    hashed = hash_with_salt(raw_key)
    key_record = find_key_by_hash(hashed)
    if not key_record:
        raise AuthenticationError("Invalid API key")
    if key_record.expires_at < utcnow():
        audit_log("api_key_expired", key_id=key_record.key_id)
        raise AuthenticationError("API key expired")
    key_record.last_used_at = utcnow()
    return key_record
```

## Access Review Queries

```sql
-- CC6: Quarterly access review — who has access and when they last used it
SELECT
    u.id,
    u.email,
    u.role,
    u.created_at,
    u.last_login_at,
    u.mfa_enabled,
    DATEDIFF(CURRENT_DATE, u.last_login_at) AS days_since_last_login
FROM users u
WHERE u.is_active = TRUE
ORDER BY u.last_login_at ASC;

-- Flag accounts inactive for 90+ days for deprovisioning review
SELECT id, email, role, last_login_at
FROM users
WHERE is_active = TRUE
  AND (last_login_at IS NULL OR last_login_at < DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY));

-- Audit: all permission changes in the last quarter
SELECT
    al.timestamp,
    al.actor_user_id,
    al.target_user_id,
    al.action,
    al.old_role,
    al.new_role
FROM audit_log al
WHERE al.action IN ('role_changed', 'user_activated', 'user_deactivated')
  AND al.timestamp >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
ORDER BY al.timestamp DESC;
```

## User Deprovisioning

```python
def deprovision_user(user_id, reason, actor_id):
    """Immediately revoke all access when an employee leaves or role changes."""
    user = get_user(user_id)

    # 1. Deactivate account
    user.is_active = False
    user.deactivated_at = utcnow()
    user.deactivated_by = actor_id
    user.deactivation_reason = reason

    # 2. Terminate all active sessions
    terminate_all_sessions(user_id)

    # 3. Revoke all API keys
    revoke_all_api_keys(user_id)

    # 4. Revoke all OAuth tokens
    revoke_all_oauth_tokens(user_id)

    # 5. Remove from all groups/teams
    remove_from_all_groups(user_id)

    audit_log("user_deprovisioned", target_user=user_id, actor=actor_id,
              reason=reason, actions=["deactivated", "sessions_terminated",
              "api_keys_revoked", "oauth_revoked", "groups_removed"])
```

## Encryption at Rest — AES-256

```python
import os
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes
import base64

def derive_key(master_key: bytes, salt: bytes) -> bytes:
    """Derive encryption key from master key using PBKDF2."""
    kdf = PBKDF2HMAC(
        algorithm=hashes.SHA256(),
        length=32,
        salt=salt,
        iterations=480000,
    )
    return base64.urlsafe_b64encode(kdf.derive(master_key))

def encrypt_field(plaintext: str, master_key: bytes) -> str:
    """Encrypt a sensitive field. Store the result in the database."""
    salt = os.urandom(16)
    key = derive_key(master_key, salt)
    f = Fernet(key)
    encrypted = f.encrypt(plaintext.encode())
    # Store salt + encrypted together
    return base64.urlsafe_b64encode(salt + encrypted).decode()

def decrypt_field(stored_value: str, master_key: bytes) -> str:
    """Decrypt a sensitive field retrieved from the database."""
    raw = base64.urlsafe_b64decode(stored_value)
    salt = raw[:16]
    encrypted = raw[16:]
    key = derive_key(master_key, salt)
    f = Fernet(key)
    return f.decrypt(encrypted).decode()

# Key rotation: decrypt with old key, encrypt with new key
def rotate_encryption_key(stored_value, old_key, new_key):
    plaintext = decrypt_field(stored_value, old_key)
    return encrypt_field(plaintext, new_key)
```

## Encryption in Transit — TLS Configuration

```python
# TLS 1.2+ enforcement — server configuration
TLS_CONFIG = {
    "minimum_version": "TLSv1.2",
    "cipher_suites": [
        "TLS_AES_256_GCM_SHA384",
        "TLS_CHACHA20_POLY1305_SHA256",
        "TLS_AES_128_GCM_SHA256",
    ],
    "hsts_max_age": 31536000,          # 1 year
    "hsts_include_subdomains": True,
    "hsts_preload": True,
}

# Internal service-to-service: always use TLS, verify certificates
import ssl

def create_internal_ssl_context():
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.minimum_version = ssl.TLSVersion.TLSv1_2
    ctx.verify_mode = ssl.CERT_REQUIRED
    ctx.load_verify_locations("/etc/ssl/internal-ca.pem")
    return ctx
```

## Secrets Management (Python)

```python
# NEVER do this:
# DATABASE_URL = "postgresql://admin:s3cr3t@db.example.com/prod"  # CC6 violation
# API_SECRET = "hardcoded-secret-value"                            # CC6 violation

# DO: Load secrets from environment or vault service
import os

def get_secret(name: str) -> str:
    """Retrieve secret from vault service, fall back to environment variable."""
    # Option 1: Vault service
    try:
        return vault_client.read(f"secret/data/{name}")["data"]["value"]
    except VaultError:
        pass

    # Option 2: Environment variable
    value = os.environ.get(name)
    if not value:
        raise ConfigurationError(f"Required secret '{name}' not configured")
    return value

DATABASE_URL = get_secret("DATABASE_URL")
API_SECRET = get_secret("API_SECRET")
```

## Secrets Management (Node.js)

```javascript
// Never commit .env files; load from vault at runtime
function getSecret(name) {
  // Try vault service first
  const vaultValue = vaultClient.read(`secret/data/${name}`);
  if (vaultValue) return vaultValue;

  // Fall back to environment
  const value = process.env[name];
  if (!value) {
    throw new Error(`Required secret '${name}' not configured`);
  }
  return value;
}

// .gitignore MUST include:
// .env
// .env.*
// *.pem
// *.key
// credentials.*
```
