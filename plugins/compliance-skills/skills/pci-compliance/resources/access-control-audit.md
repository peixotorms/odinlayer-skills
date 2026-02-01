# Access Control & Audit Logging Patterns

Code examples for access control and audit logging referenced from the main PCI DSS compliance skill (Sections 6 and 7).

## Access Control Patterns

### Role-Based Access Control

Only authorized roles should access cardholder data endpoints. Implement middleware that enforces this before the request reaches your handler.

**Node/Express Middleware:**
```javascript
const PAYMENT_ROLES = ['payment_admin', 'billing_manager'];

function requirePaymentAccess(req, res, next) {
    const user = req.auth; // Set by authentication middleware

    if (!user) {
        auditLog('payment_access_denied', {
            reason: 'unauthenticated',
            ip: req.ip,
            path: req.path,
        });
        return res.status(401).json({ error: 'Authentication required' });
    }

    if (!PAYMENT_ROLES.includes(user.role)) {
        auditLog('payment_access_denied', {
            userId: user.id,
            role: user.role,
            ip: req.ip,
            path: req.path,
        });
        return res.status(403).json({ error: 'Insufficient permissions' });
    }

    auditLog('payment_access_granted', {
        userId: user.id,
        role: user.role,
        ip: req.ip,
        path: req.path,
    });
    next();
}

// Apply to all payment routes
app.use('/api/payment', requirePaymentAccess);
```

**Python/Flask Decorator:**
```python
from functools import wraps
from flask import request, jsonify, g

PAYMENT_ROLES = {'payment_admin', 'billing_manager'}

def require_payment_access(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        user = g.get('current_user')

        if not user:
            audit_log('payment_access_denied', {
                'reason': 'unauthenticated',
                'ip': request.remote_addr,
                'path': request.path,
            })
            return jsonify({'error': 'Authentication required'}), 401

        if user.role not in PAYMENT_ROLES:
            audit_log('payment_access_denied', {
                'user_id': user.id,
                'role': user.role,
                'ip': request.remote_addr,
                'path': request.path,
            })
            return jsonify({'error': 'Insufficient permissions'}), 403

        audit_log('payment_access_granted', {
            'user_id': user.id,
            'role': user.role,
            'ip': request.remote_addr,
            'path': request.path,
        })
        return f(*args, **kwargs)
    return decorated

@app.route('/api/payment/charge', methods=['POST'])
@require_payment_access
def charge():
    pass
```

**PHP Middleware:**
```php
class PaymentAccessMiddleware
{
    private const PAYMENT_ROLES = ['payment_admin', 'billing_manager'];

    public function handle(Request $request, Closure $next)
    {
        $user = $request->user();

        if (!$user) {
            AuditLog::write('payment_access_denied', [
                'reason' => 'unauthenticated',
                'ip'     => $request->ip(),
                'path'   => $request->path(),
            ]);
            return response()->json(['error' => 'Authentication required'], 401);
        }

        if (!in_array($user->role, self::PAYMENT_ROLES, true)) {
            AuditLog::write('payment_access_denied', [
                'user_id' => $user->id,
                'role'    => $user->role,
                'ip'      => $request->ip(),
                'path'    => $request->path(),
            ]);
            return response()->json(['error' => 'Insufficient permissions'], 403);
        }

        AuditLog::write('payment_access_granted', [
            'user_id' => $user->id,
            'role'    => $user->role,
            'ip'      => $request->ip(),
            'path'    => $request->path(),
        ]);

        return $next($request);
    }
}
```

### Database Permission Separation

```sql
-- Payment service account: minimal permissions
CREATE USER 'payment_svc'@'localhost' IDENTIFIED BY '...';
GRANT SELECT, INSERT ON payments.transactions TO 'payment_svc'@'localhost';
GRANT SELECT ON payments.token_vault TO 'payment_svc'@'localhost';
-- NO DELETE, NO DROP, NO ALTER, NO access to other schemas

-- Application service account: no access to payment tables
CREATE USER 'app_svc'@'localhost' IDENTIFIED BY '...';
GRANT SELECT, INSERT, UPDATE ON app.users TO 'app_svc'@'localhost';
-- NO access to payments schema at all
```

## Audit Logging

### What to Log

Every interaction with cardholder data must produce an audit record containing:

| Field | Description | Example |
|-------|-------------|---------|
| `timestamp` | ISO 8601 with timezone | `2026-01-15T14:30:00.000Z` |
| `event_type` | Action category | `payment_attempt`, `detokenize_request` |
| `user_id` | Authenticated user identifier | `usr_abc123` |
| `ip_address` | Source IP of the request | `203.0.113.42` |
| `resource` | What was accessed | `/api/payment/charge` |
| `action` | Specific operation | `create_payment_intent` |
| `result` | Outcome | `success`, `denied`, `error` |
| `metadata` | Additional safe context | `{"intent_id": "pi_xxx", "amount": 2000}` |

### What NEVER to Log

```
PROHIBITED in any log output:
  - Full PAN (card number)
  - CVV / CVC / CID
  - PIN or PIN block
  - Full magnetic stripe data
  - Encryption keys
  - Passwords or password hashes
  - Full expiration date combined with other card data
```

### Structured Audit Logger

**Node.js:**
```javascript
const crypto = require('crypto');

class AuditLogger {
    constructor(storage) {
        this._storage = storage;
        this._previousHash = null;
    }

    log(eventType, details) {
        const entry = {
            id: crypto.randomUUID(),
            timestamp: new Date().toISOString(),
            event_type: eventType,
            user_id: details.userId || null,
            ip_address: details.ip || null,
            resource: details.resource || null,
            action: details.action || null,
            result: details.result || null,
            metadata: this._sanitize(details.metadata || {}),
        };

        // Hash chain for tamper evidence
        const payload = JSON.stringify(entry) + (this._previousHash || '');
        entry.chain_hash = crypto
            .createHash('sha256')
            .update(payload)
            .digest('hex');
        this._previousHash = entry.chain_hash;

        // Append-only write
        this._storage.append(entry);
        return entry;
    }

    _sanitize(metadata) {
        const sanitized = { ...metadata };
        // Remove any accidentally included card data
        for (const [key, value] of Object.entries(sanitized)) {
            if (typeof value === 'string') {
                sanitized[key] = sanitizeLogEntry(value);
            }
        }
        return sanitized;
    }
}
```

**Python:**
```python
import hashlib
import json
import uuid
from datetime import datetime, timezone

class AuditLogger:
    def __init__(self, storage):
        self._storage = storage
        self._previous_hash = None

    def log(self, event_type: str, **details) -> dict:
        entry = {
            'id': str(uuid.uuid4()),
            'timestamp': datetime.now(timezone.utc).isoformat(),
            'event_type': event_type,
            'user_id': details.get('user_id'),
            'ip_address': details.get('ip'),
            'resource': details.get('resource'),
            'action': details.get('action'),
            'result': details.get('result'),
            'metadata': self._sanitize(details.get('metadata', {})),
        }

        # Tamper-evident hash chain
        payload = json.dumps(entry, sort_keys=True)
        if self._previous_hash:
            payload += self._previous_hash
        entry['chain_hash'] = hashlib.sha256(payload.encode()).hexdigest()
        self._previous_hash = entry['chain_hash']

        self._storage.append(entry)
        return entry

    def _sanitize(self, metadata: dict) -> dict:
        sanitized = {}
        for key, value in metadata.items():
            if isinstance(value, str):
                sanitized[key] = sanitize_log_entry(value)
            else:
                sanitized[key] = value
        return sanitized
```

### Log Retention

PCI DSS requires audit logs to be retained for a minimum of **one year**, with at least **three months immediately available** for analysis. Configure log rotation and archival accordingly:

```
Log storage requirements:
  - Minimum retention: 12 months
  - Immediately available: 3 months
  - Append-only storage (no modification or deletion)
  - Centralized collection from all payment components
  - Access to logs restricted to security/audit roles only
```
