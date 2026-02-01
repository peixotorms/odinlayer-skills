# Secure Coding Practices

Code examples for secure coding patterns referenced from the main PCI DSS compliance skill (Section 8).

## Input Validation

Validate all payment-related input at the boundary. Reject malformed data before it reaches business logic.

**Card Number Validation (Node.js):**
```javascript
function validateCardInput(input) {
    const errors = [];

    // PAN: strip spaces/dashes, check length and Luhn
    const pan = (input.cardNumber || '').replace(/[\s-]/g, '');
    if (!/^\d{13,19}$/.test(pan)) {
        errors.push('Card number must be 13-19 digits');
    } else if (!luhnCheck(pan)) {
        errors.push('Invalid card number');
    }

    // Expiry: MM/YY format, not in the past
    const expiry = input.expiry || '';
    if (!/^\d{2}\/\d{2}$/.test(expiry)) {
        errors.push('Expiry must be MM/YY format');
    } else {
        const [month, year] = expiry.split('/').map(Number);
        const expDate = new Date(2000 + year, month, 0); // Last day of month
        if (expDate < new Date()) {
            errors.push('Card is expired');
        }
    }

    // CVV: 3 digits (Visa/MC/Discover) or 4 digits (Amex)
    const cvv = input.cvv || '';
    if (!/^\d{3,4}$/.test(cvv)) {
        errors.push('CVV must be 3 or 4 digits');
    }

    return { valid: errors.length === 0, errors };
}
```

## Parameterized Queries

Never concatenate card data or any user input into SQL strings.

**WRONG -- SQL injection risk:**
```python
# NEVER DO THIS
cursor.execute(
    f"SELECT * FROM transactions WHERE card_token = '{token}'"
)
```

**CORRECT -- parameterized query:**
```python
cursor.execute(
    "SELECT * FROM transactions WHERE card_token = %s",
    (token,)
)
```

**CORRECT -- ORM (SQLAlchemy):**
```python
transaction = db.session.query(Transaction).filter(
    Transaction.card_token == token
).first()
```

**CORRECT -- PHP PDO:**
```php
$stmt = $pdo->prepare('SELECT * FROM transactions WHERE card_token = :token');
$stmt->execute(['token' => $token]);
$transaction = $stmt->fetch();
```

## Error Handling

Never expose card data or internal system details in error messages.

**WRONG:**
```javascript
// Leaks card number in error response
catch (err) {
    res.status(500).json({
        error: `Payment failed for card ${cardNumber}: ${err.message}`,
        stack: err.stack,
    });
}
```

**CORRECT:**
```javascript
catch (err) {
    const errorId = crypto.randomUUID();

    // Log full details internally (with sanitization)
    auditLog('payment_error', {
        errorId,
        message: sanitizeLogEntry(err.message),
        userId: req.auth.id,
        ip: req.ip,
    });

    // Return safe error to client
    res.status(500).json({
        error: 'Payment processing failed',
        errorId, // Reference ID for support
    });
}
```

## Memory Handling

Clear sensitive data from memory after use to minimize exposure window.

**Node.js -- Buffer clearing:**
```javascript
function processCardSecurely(cardNumber) {
    const buffer = Buffer.from(cardNumber, 'utf8');
    try {
        // Process the card data
        const token = tokenize(buffer.toString());
        return token;
    } finally {
        // Zero out the buffer
        buffer.fill(0);
    }
}
```

**Python -- Memory clearing:**
```python
import ctypes

def secure_clear(s: str):
    """Attempt to overwrite string in memory."""
    if not isinstance(s, str):
        return
    ptr = ctypes.cast(id(s), ctypes.POINTER(ctypes.c_char))
    for i in range(len(s)):
        ptr[ctypes.sizeof(ctypes.c_ssize_t) * 2 + i] = b'\x00'

def process_card_securely(card_number: str) -> str:
    try:
        token = tokenize(card_number)
        return token
    finally:
        secure_clear(card_number)
```

**PHP -- Unset and overwrite:**
```php
function processCardSecurely(string &$cardNumber): string
{
    try {
        $token = tokenize($cardNumber);
        return $token;
    } finally {
        // Overwrite the variable content before unsetting
        $cardNumber = str_repeat("\0", strlen($cardNumber));
        unset($cardNumber);
    }
}
```
