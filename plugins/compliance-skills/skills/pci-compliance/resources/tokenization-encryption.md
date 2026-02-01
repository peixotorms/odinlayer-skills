# Tokenization & Encryption Patterns

Code examples for tokenization and encryption referenced from the main PCI DSS compliance skill (Sections 4 and 5).

## Tokenization Patterns

### Preferred: Use Payment Processor Tokens

The safest PCI strategy is to never let card data touch your servers. Use client-side tokenization provided by your payment processor.

**Stripe Elements (client-side JavaScript):**
```javascript
// Card data goes directly from user's browser to Stripe
// Your server never sees the raw card number
const stripe = Stripe('pk_live_...');
const elements = stripe.elements();
const cardElement = elements.create('card');
cardElement.mount('#card-element');

async function handlePayment(event) {
    event.preventDefault();

    const { paymentMethod, error } = await stripe.createPaymentMethod({
        type: 'card',
        card: cardElement,
    });

    if (error) {
        showError(error.message);
        return;
    }

    // Send only the token/paymentMethod ID to your server
    const response = await fetch('/api/charge', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ paymentMethodId: paymentMethod.id }),
    });
}
```

**Server-side PaymentIntent (Node.js) -- only handles tokens:**
```javascript
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

app.post('/api/charge', async (req, res) => {
    const { paymentMethodId } = req.body;

    // Server only receives the token, never the card number
    const paymentIntent = await stripe.paymentIntents.create({
        amount: 2000, // $20.00 in cents
        currency: 'usd',
        payment_method: paymentMethodId,
        confirm: true,
        return_url: 'https://example.com/return',
    });

    auditLog('payment_attempt', {
        intentId: paymentIntent.id,
        status: paymentIntent.status,
        // NEVER log: card number, CVV, expiry
    });

    res.json({ status: paymentIntent.status });
});
```

**Server-side PaymentIntent (PHP):**
```php
$stripe = new \Stripe\StripeClient(getenv('STRIPE_SECRET_KEY'));

$paymentIntent = $stripe->paymentIntents->create([
    'amount' => 2000,
    'currency' => 'usd',
    'payment_method' => $request->input('paymentMethodId'),
    'confirm' => true,
    'return_url' => 'https://example.com/return',
]);

auditLog('payment_attempt', [
    'intentId' => $paymentIntent->id,
    'status' => $paymentIntent->status,
]);
```

### Custom Tokenization (When Required)

If you must build your own tokenization layer (rare -- strongly prefer processor tokens):

**Token Vault Pattern (Python):**
```python
import os
import hashlib
from cryptography.fernet import Fernet

class TokenVault:
    def __init__(self, encryption_key: bytes):
        self._fernet = Fernet(encryption_key)

    def tokenize(self, pan: str) -> str:
        """Replace PAN with a random token. Store encrypted mapping."""
        token = os.urandom(16).hex()  # Random, non-reversible token
        encrypted_pan = self._fernet.encrypt(pan.encode())

        # Store in database: token → encrypted_pan
        db.execute(
            "INSERT INTO token_vault (token, encrypted_pan, created_at) "
            "VALUES (?, ?, NOW())",
            (token, encrypted_pan)
        )

        audit_log('pan_tokenized', {'token': token, 'pan_last4': pan[-4:]})
        return token

    def detokenize(self, token: str, requesting_user: str) -> str:
        """Retrieve PAN from token. Requires authorization and logs access."""
        audit_log('detokenize_request', {
            'token': token,
            'user': requesting_user
        })

        row = db.execute(
            "SELECT encrypted_pan FROM token_vault WHERE token = ?",
            (token,)
        ).fetchone()

        if not row:
            raise ValueError("Invalid token")

        pan = self._fernet.decrypt(row['encrypted_pan']).decode()

        audit_log('detokenize_success', {
            'token': token,
            'user': requesting_user,
            'pan_last4': pan[-4:]
        })
        return pan
```

## Encryption Requirements

### Encryption at Rest -- AES-256-GCM

All stored cardholder data must be encrypted with strong cryptography. AES-256-GCM provides both confidentiality and integrity.

**PHP (OpenSSL):**
```php
function encryptCardData(string $plaintext, string $key): string
{
    // Key must be 32 bytes for AES-256
    if (strlen($key) !== 32) {
        throw new InvalidArgumentException('Key must be 32 bytes');
    }
    $iv = random_bytes(12); // 96-bit IV for GCM
    $tag = '';
    $ciphertext = openssl_encrypt(
        $plaintext,
        'aes-256-gcm',
        $key,
        OPENSSL_RAW_DATA,
        $iv,
        $tag,
        '', // additional authenticated data
        16  // tag length
    );
    // Store IV + tag + ciphertext together
    return base64_encode($iv . $tag . $ciphertext);
}

function decryptCardData(string $encoded, string $key): string
{
    $data = base64_decode($encoded);
    $iv = substr($data, 0, 12);
    $tag = substr($data, 12, 16);
    $ciphertext = substr($data, 28);
    $plaintext = openssl_decrypt(
        $ciphertext,
        'aes-256-gcm',
        $key,
        OPENSSL_RAW_DATA,
        $iv,
        $tag
    );
    if ($plaintext === false) {
        throw new RuntimeException('Decryption failed -- data may be tampered');
    }
    return $plaintext;
}
```

**Python (cryptography library):**
```python
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_card_data(plaintext: str, key: bytes) -> bytes:
    """Encrypt with AES-256-GCM. Key must be 32 bytes."""
    if len(key) != 32:
        raise ValueError("Key must be 32 bytes for AES-256")
    nonce = os.urandom(12)  # 96-bit nonce
    aesgcm = AESGCM(key)
    ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)
    return nonce + ciphertext  # Prepend nonce for storage

def decrypt_card_data(data: bytes, key: bytes) -> str:
    """Decrypt AES-256-GCM. First 12 bytes are nonce."""
    nonce = data[:12]
    ciphertext = data[12:]
    aesgcm = AESGCM(key)
    plaintext = aesgcm.decrypt(nonce, ciphertext, None)
    return plaintext.decode()
```

### Encryption in Transit -- TLS 1.2+

All cardholder data must be transmitted over TLS 1.2 or higher. Never transmit card data over plain HTTP.

**HTTP to HTTPS Redirect (Node/Express):**
```javascript
// Force HTTPS on all payment routes
app.use('/api/payment', (req, res, next) => {
    if (req.headers['x-forwarded-proto'] !== 'https' && !req.secure) {
        return res.redirect(301, `https://${req.hostname}${req.originalUrl}`);
    }
    next();
});
```

**Security Headers (Node/Express):**
```javascript
app.use((req, res, next) => {
    // HSTS: enforce HTTPS for 1 year, include subdomains
    res.setHeader(
        'Strict-Transport-Security',
        'max-age=31536000; includeSubDomains; preload'
    );
    // Prevent framing (clickjacking protection for payment pages)
    res.setHeader('X-Frame-Options', 'DENY');
    // Prevent MIME type sniffing
    res.setHeader('X-Content-Type-Options', 'nosniff');
    // Content Security Policy for payment pages
    res.setHeader(
        'Content-Security-Policy',
        "default-src 'self'; script-src 'self' https://js.stripe.com; frame-src https://js.stripe.com;"
    );
    next();
});
```

**Secure Cookie Flags (PHP):**
```php
// All cookies on payment pages must use these flags
session_set_cookie_params([
    'lifetime' => 0,
    'path'     => '/',
    'domain'   => '.example.com',
    'secure'   => true,       // Only sent over HTTPS
    'httponly'  => true,       // Not accessible via JavaScript
    'samesite' => 'Strict',   // Prevent CSRF
]);
session_start();
```

**Secure Cookie Flags (Node/Express):**
```javascript
app.use(session({
    secret: process.env.SESSION_SECRET,
    cookie: {
        secure: true,       // HTTPS only
        httpOnly: true,     // No JS access
        sameSite: 'strict', // CSRF protection
        maxAge: 3600000,    // 1 hour
    },
    resave: false,
    saveUninitialized: false,
}));
```

### Key Management

```
NEVER:
  - Hardcode encryption keys in source code
  - Store keys alongside the data they encrypt
  - Use the same key for encryption and authentication
  - Commit keys or secrets to version control

ALWAYS:
  - Load keys from environment variables or a secrets manager
  - Rotate keys at least annually
  - Use separate keys for separate data types
  - Implement key versioning to support rotation
```

**Key Rotation Pattern (Python):**
```python
import os
import json

class KeyManager:
    def __init__(self):
        # Keys loaded from environment or secrets manager
        self._keys = {
            'v2': os.environ['ENCRYPTION_KEY_V2'].encode(),
            'v1': os.environ['ENCRYPTION_KEY_V1'].encode(),
        }
        self._current_version = 'v2'

    def encrypt(self, plaintext: str) -> dict:
        key = self._keys[self._current_version]
        ciphertext = encrypt_card_data(plaintext, key)
        return {
            'key_version': self._current_version,
            'data': ciphertext.hex(),
        }

    def decrypt(self, envelope: dict) -> str:
        version = envelope['key_version']
        key = self._keys[version]
        return decrypt_card_data(bytes.fromhex(envelope['data']), key)
```
