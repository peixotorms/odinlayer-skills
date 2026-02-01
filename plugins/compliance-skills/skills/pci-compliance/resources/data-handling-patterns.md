# Data Handling Patterns

Code examples for PAN masking, log sanitization, and Luhn validation referenced from the main PCI DSS compliance skill (Section 3: Data Classification).

## PAN Masking Function

Display only the first 6 and last 4 digits of a card number; mask the rest with asterisks.

**PHP:**
```php
function maskPan(string $pan): string
{
    $cleaned = preg_replace('/\D/', '', $pan);
    $len = strlen($cleaned);
    if ($len < 13 || $len > 19) {
        return str_repeat('*', $len);
    }
    $first6 = substr($cleaned, 0, 6);
    $last4 = substr($cleaned, -4);
    $masked = str_repeat('*', $len - 10);
    return $first6 . $masked . $last4;
}
// "4111111111111111" → "411111******1111"
```

**Python:**
```python
import re

def mask_pan(pan: str) -> str:
    cleaned = re.sub(r'\D', '', pan)
    length = len(cleaned)
    if length < 13 or length > 19:
        return '*' * length
    return cleaned[:6] + '*' * (length - 10) + cleaned[-4:]

# "4111111111111111" → "411111******1111"
```

**JavaScript/Node:**
```javascript
function maskPan(pan) {
    const cleaned = pan.replace(/\D/g, '');
    const len = cleaned.length;
    if (len < 13 || len > 19) {
        return '*'.repeat(len);
    }
    return cleaned.slice(0, 6) + '*'.repeat(len - 10) + cleaned.slice(-4);
}
// "4111111111111111" → "411111******1111"
```

## Log Sanitization

Regex patterns to detect and redact card numbers from log output before writing.

**PHP:**
```php
function sanitizeLogEntry(string $message): string
{
    // Match 13-19 digit sequences that pass Luhn check
    return preg_replace_callback(
        '/\b(\d{13,19})\b/',
        function ($matches) {
            if (luhnCheck($matches[1])) {
                return maskPan($matches[1]);
            }
            return $matches[1];
        },
        $message
    );
}

// Also catch formatted card numbers with spaces or dashes
function sanitizeFormattedCards(string $message): string
{
    return preg_replace(
        '/\b(\d{4}[\s\-]?\d{4}[\s\-]?\d{4}[\s\-]?\d{1,7})\b/',
        '[CARD_REDACTED]',
        $message
    );
}
```

**Python:**
```python
import re

CARD_PATTERN = re.compile(r'\b(\d{13,19})\b')
FORMATTED_CARD = re.compile(r'\b(\d{4}[\s\-]?\d{4}[\s\-]?\d{4}[\s\-]?\d{1,7})\b')

def sanitize_log_entry(message: str) -> str:
    def replace_match(match):
        digits = match.group(1)
        if luhn_check(digits):
            return mask_pan(digits)
        return digits
    message = CARD_PATTERN.sub(replace_match, message)
    message = FORMATTED_CARD.sub('[CARD_REDACTED]', message)
    return message
```

**JavaScript/Node:**
```javascript
const CARD_PATTERN = /\b(\d{13,19})\b/g;
const FORMATTED_CARD = /\b(\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{1,7})\b/g;

function sanitizeLogEntry(message) {
    message = message.replace(CARD_PATTERN, (match) => {
        return luhnCheck(match) ? maskPan(match) : match;
    });
    return message.replace(FORMATTED_CARD, '[CARD_REDACTED]');
}
```

## Luhn Validation

Use to detect whether a digit sequence is a card number (for log sanitization and input validation).

**PHP:**
```php
function luhnCheck(string $number): bool
{
    $cleaned = preg_replace('/\D/', '', $number);
    $len = strlen($cleaned);
    if ($len < 13 || $len > 19) {
        return false;
    }
    $sum = 0;
    $alternate = false;
    for ($i = $len - 1; $i >= 0; $i--) {
        $digit = (int) $cleaned[$i];
        if ($alternate) {
            $digit *= 2;
            if ($digit > 9) {
                $digit -= 9;
            }
        }
        $sum += $digit;
        $alternate = !$alternate;
    }
    return $sum % 10 === 0;
}
```

**Python:**
```python
def luhn_check(number: str) -> bool:
    cleaned = re.sub(r'\D', '', number)
    if not (13 <= len(cleaned) <= 19):
        return False
    total = 0
    alternate = False
    for ch in reversed(cleaned):
        digit = int(ch)
        if alternate:
            digit *= 2
            if digit > 9:
                digit -= 9
        total += digit
        alternate = not alternate
    return total % 10 == 0
```

**JavaScript/Node:**
```javascript
function luhnCheck(number) {
    const cleaned = number.replace(/\D/g, '');
    if (cleaned.length < 13 || cleaned.length > 19) return false;
    let sum = 0;
    let alternate = false;
    for (let i = cleaned.length - 1; i >= 0; i--) {
        let digit = parseInt(cleaned[i], 10);
        if (alternate) {
            digit *= 2;
            if (digit > 9) digit -= 9;
        }
        sum += digit;
        alternate = !alternate;
    }
    return sum % 10 === 0;
}
```
